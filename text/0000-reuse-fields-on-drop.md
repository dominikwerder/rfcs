- Feature Name: `reuse_fields_on_drop`
- Start Date: 2018-10-19
- RFC PR: …
- Rust Issue: …

# Pre-RFC

# Summary
[summary]: #summary

The fields which a `struct` is made of should be re-usable on drop.
Currently, we can not move out any fields in `drop(&mut self)`.
Work-arounds include to use `Option` or `ManuallyDrop` as fields and move *their* content in `drop(&mut self)`.
Both approaches have [issues](#issues).
After I started the topic in
[here](https://internals.rust-lang.org/t/re-use-struct-fields-on-drop-was-drop-mut-self-vs-drop-self/8594)
after which the discussion moved on to a
[second](https://internals.rust-lang.org/t/making-drop-more-magic-to-make-it-less-magic/8612) thread.
This pre-RFC starts with the illustration of the current situation, and continues with a [possible
solution](#possible-solution-1).


# Motivation
[motivation]: #motivation

## Issue with today's Rust
[issues]: #issues

### Example 1

As an example to illustrate the issue, imagine a `struct Writer` which owns a `File`.  When `Writer` gets
dropped, we want to make sure that it sends that `File` elsewhere for further usage.

```rust
struct Writer {
  file: File,
  sender: Sender<File>,
}

impl Drop for Writer {
  fn drop(&mut self) {
    // DOES NOT WORK:
    self.sender.send(self.file).unwrap();
  }
}
```

This does not work because we can not move `file` or `sender` out of `&mut self`.


### Example 2

A straight-forward workaround is to put the fields that we would like to move into `Option`.
This works, but has disadvantages:

```rust
struct Writer {
  // BAD: All member-functions have to deal with a possible `None` even though
  // we never intend to construct a `Writer` without a `File`.
  file: Option<File>,
  sender: Option<Sender<File>>,
}

impl Drop for Writer {
  fn drop(&mut self) {
    match self.sender.take() {
      Some(sender) => match self.file.take() {
        Some(file) => sender.send(file).expect("send failed"),
        None => panic!("That should never happen")
      }
      None => panic!("That should never happen")
    }
  }
}

impl Writer {
  fn write_something(&mut self) {
    // BAD: Runtime cost for each method call, and unnecessary additional
    // sad path
    match &mut self.file {
      Some(file) => {
        file.write(b"data")
      }
      None => panic!("That should never happen")
    }
  }
}
```

Example 2 allows us to re-use `File`, but at what cost:

1) We essentially introduced an additional null-like value for the type `Writer` which serves absolutely no
   purpose, except for enabling us to move out in `drop`.  This is very ugly, especially if we value the
   principle of making useless/disfunctional state irrepresentable.
2) We introduced many additional error paths which were not necessary in example 1
3) We introduced a runtime cost for each call to a member function which wants to
   use the content of `self.file`


### Example 3

A zero-cost work-around which also works on today's Rust would be:

```rust
struct Writer {
  /*
  GOOD: Member functions don't have to check for `None`.
  BAD: Member functions can see that we use `ManuallyDrop`.  This is not a huge deal, but unnecessary.  We
  can separate the concerns better as we see below in the proposed solution.
  */
  file: ManuallyDrop<File>,
  sender: ManuallyDrop<Sender<File>>,
}

impl Drop for Writer {
  fn drop(&mut self) {
    // BAD: Have to invoke `unsafe`.
    let sender = unsafe { ManuallyDrop::into_inner(ptr::read(&self.sender)) };
    let file = unsafe { ManuallyDrop::into_inner(ptr::read(&self.file)) };
    sender.send(file).unwrap();
  }
}

impl Writer {
  fn new(file: File, sender: Sender<File>) -> Self {
    Self {
      file: ManuallyDrop::new(file),
      sender: ManuallyDrop::new(sender),
    }
  }

  fn write_something(&mut self) {
    // BAD: Runtime cost for each method call, and unnecessary additional
    // sad path
    match &mut self.file {
      Some(file) => {
        file.write(b"data")
      }
      None => panic!("That should never happen")
    }
  }
}
```

Example 3 allows us to re-use `File`, similar to example 2, but with the additional advantage that member
functions don't need to deal with a possible `None` in the fields.  This reduces error paths and runtime
cost.


### Example 4

With some straight-forward `proc-macro` we could reduce the boilerplate.
The proc macro recognizes the optional `#[rescue_me]` attributes on the fields.

```rust
#[derive(RescueOnDrop)]
#[rescue_with="Writer::rescue"]
#[rescue_before="Writer::do_before_rescue"]
struct Writer {
  /*
  GOOD: Member functions don't have to check for `None`.
  BAD: Member functions can see that we use `ManuallyDrop`.  This is not a huge deal, but unnecessary.  We
  can separate the concerns better as we see below in the proposed solution.
  */
  #[rescue_me]  file: ManuallyDrop<File>,
  #[rescue_me]  sender: ManuallyDrop<Sender<File>>,
  this_will_not_be_rescued: UnimportantType,
}

impl Writer {
  fn new(file: File, sender: Sender<File>) -> Self {
    Self {
      file: ManuallyDrop::new(file),
      sender: ManuallyDrop::new(sender),
    }
  }

  fn write_something(&mut self) {
    // GOOD: No check for `None`.
    file.write(b"data");
  }

  fn rescue(file: File, sender: Sender<File>) {
    sender.send(file).expect("sending failed");
  }

  fn do_before_rescue(&mut self) {
    // put safe things here which normally would happen in `drop(&mut self)`
  }
}

// BEGIN  derive(RescueOnDrop) automatically generates:
impl Drop for Writer {
  fn drop(&mut self) {
    Writer::do_before_rescue(self);
    let file = unsafe { ManuallyDrop::into_inner(ptr::read(&self.file)) };
    let sender = unsafe { ManuallyDrop::into_inner(ptr::read(&self.sender)) };
    Writer::rescue(file, sender);
  }
}
// END of autogenerated code.
```



Still, the we could do better:


# Possible solution 1

This proposal is *fully* backwards compatible and does *not* change any `Drop` behavior.

Proposal:

- Add a new field attribute `#[rescue_me]` (naming this attribute is totally up for discussion).
- Let the compiler call `drop(&mut self)` as usual.
- After `drop(&mut self)` returned, if the type implements `RescueOnDrop`, let the compiler destructure the
  `struct` and pass the rescued fields to the specified function (`Writer::rescue` in this example).

```rust
#[derive(RescueOnDrop)]
#[rescue_with="Writer::rescue"]
struct Writer {
  // GOOD:  No `ManuallyDrop` needed.
  #[rescue_me]  file: File,
  #[rescue_me]  sender: Sender<File>,
  this_will_not_be_rescued: UnimportantType,
}

impl Writer {
  fn new(file: File, sender: Sender<File>) -> Self {
    // GOOD:  Simple, nothing special here.
    Self {
      file,
      sender,
    }
  }

  fn write_something(&mut self) {
    // GOOD: No check for `None`.
    file.write(b"data");
  }

  fn rescue(file: File, sender: Sender<File>) {
    // GOOD: Can do something useful with the fields.
    sender.send(file).expect("sending failed");
  }
}

impl Drop for Writer {
  fn drop(&mut self) {
    // Can do safe things here with the as of yet still functional `&mut self: &mut Writer`
  }
}
```



# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

TODO.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TODO.


# Drawbacks
[drawbacks]: #drawbacks

TODO.


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

TODO.

- Best safety
- Best separation of concerns
- Least boilerplate
- Fully backwards compatible


# Prior art
[prior-art]: #prior-art

TODO.


# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Choosing names for any new attribute or macro?
- Can one handle `enum` as well?
