Rust
====

## Installation

- `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`
  - this downloads and runs `rustup-init.sh`, which downloads and runs
    <https://static.rust-lang.org/rustup/dist/x86_64-unknown-linux-gnu/rustup-init>
- `rustup-init`
  - copies itself as `rustup` to `~/.cargo/bin`, or `$CARGO_HOME/bin`
    - yes, `rustup-init` is installed as `rustup` itself
  - creates hardlinks such as `cargo`, `rustc`, etc under `$CARGO_HOME/bin`
    - yes, toolchain binaries are hardlinks to `rustup`
    - when executed, they find the binary of the same name under
      `$RUSTUP_HOME/toolchains/stable-x86_64-unknown-linux-gnu/bin` and execve
  - runs `rustup` to install these components to `~/.rustup`, or `$RUSTUP_HOME`
    - cargo
    - clippy
    - rust-docs
    - rust-std
    - rustc
    - rustfmt
  - edit `.profile` to source `$CARGO_HOME/env`
    - `$CARGO_HOME/env` adds `$CARGO_HOME/bin` to `$PATH`

## Cross-Compiler

- `rustup target add aarch64-unknown-linux-gnu`
- `cargo build --target aarch64-unknown-linux-gnu`
- edit `~/.cargo/config` to add
    [target.aarch64-unknown-linux-gnu]
    linker = "aarch64-linux-gnu-gcc"

## cargo

- `~/.cargo` or `$CARGO_HOME`
- `$CARGO_HOME/registry`
  - `index` appears to be
    `git clone --bare https://github.com/rust-lang/crates.io-index.git`
  - `cache` contains downloaded `.crate` files (can be renamed to `.tar.gz`)
  - `src` contains un-tared crates
- `cargo build`
  - downloads dependencies to `$CARGO_HOME/registry/src`
  - builds dependencies in `target/debug/deps`


## Ownership

- each value in Rust has a variable that is the owner of the value
- there can only be one owner for a value at any time
- when the owner goes out of scope, the value is dropped
- ownership can be transferred from one variable to another
  - `let a = String::from("hello"); let b = a;`
  - b becomes the owner of the String
- when a variable is mutable, it can own different values at different times
  - `let mut a = 3; a = 4`
- simple types, where ownership transfer may be more expensive than copying,
  usually implement Copy trait
  - `let a = 3; let b = a;`
  - both a and b own different values
  - no ownership transfer
- ownerhsip eliminates one major issue: the value of a variable can be
  modified behind the back of the variable
  - no more dangling pointer like in C/C++

## Borrowing

- References for a variable can be created.  They "borrow" the onwership from
  the variable.
  - there are also "unsafe" pointers for a variable
- a variable's ownership of a value can be immutably borrowed
  - `let a = 3; let b = &a; let c = &a;`
  - there can be multiple immutable borrowers
  - until b and c go out of scope, a's ownership is immutably borrowed.  a can
    still be used, but its ownership of the value cannot be transferred
- a variable's ownership of a value can also be mutably borrowed
  - `let mut a = 3; let b = &mut a;`
  - there can only be one mutable borrower
  - until b goes out of scope, a's ownership is mutably borrowed.  a cannot be
    used.
- a variable cannot be borrowed mutably and immutably at the same time
- Internally, a reference is a pointer to the original value.  The borrowing
  rules make sure there is no surprise

## Arrays and Slices

- arrays have type `[T; N]`; different Ns mean different types
- slices have type `[T]`
  - slices have unknown size.  Similar to `str` or `Path`, there can only be
    references to slices
  - a slice borrows from an aray (or some collections)
- vectors implements trait `Deref<Target = [T]>`, meaning they can be treated
  like slices
  - looping vectors in for-loops moves because of `into_iter`
  - looping slices in for-loops copies because references implement Copy trait
- Internally, a slice is two pointers, delimiting a region in the original
  value.  The borrowing rules maek sure there is no surprise.
  - this is why slices are for types that are contiguous in memory

## Pattern Matching

- when a pattern has a match, a variable binding can be optionally created
  - `Some(val)` creates a variable binding which becomes the owner of the
    value (unless the type has Copy trait)
  - `Some(_)` does not
  - `Some(mut val)` creates a mutable variable binding
  - `Some(ref val)` creates a reference
  - `Some(ref mut val)` creates a mutable reference
  - `Some(&val)` can match `Option<&T>` and val becomes the owner of `T`
    - T should be a reference itself because we cannot move a borrowed
      ownership

## Iterators

- a for-loopable variable must implement IntoIter trait
  - the trait allows a variable to be turned into a iterator and transfers the
    ownership to the iterator
- a for-loop is a syntax sugar that turns the looped variable into an iterator
  - `for v in var` becomes roughly
    - `let mut iter = var.into_iter();`
    - `while let v = Some(var.next())`

## Interior Mutability

- interior mutability is similar to a mutable data member in a C++ class
- `UnsafeCell<T>` is T, but provides a get() method to return `*mut T`
  - it is unsafe (because a pointer to T does not borrow the ownership)
  - there is no additional cost
- `Cell<T>` is T with the interior mutability
  - `let a = Cell::new(3); a.set(4); let b = a.get();`
  - it is implemented on top of `UnsafeCell<T>`
  - it is safe because get() returns a copy of T
  - there is no additional cost
- `RefCell<T>` is T with the interior mutability
  - `let a = RefCell::new(3); *a.borrow_mut() = 4; let b = a.borrow();`
  - there is a cost to track the borrow counts dynamically

## Smart Pointers

- `Box<T>` is T put on the heap
  - T's size can be unknown (e.g., recursive types)
    - `struct List<T> { node: T, next: Box<List<T>> }`
  - like `std::unique_ptr<T>`
- `Rc<T>` is T put on the heap together with a refcount
  - like `std::shared_ptr<T>`
- `Arc<T>` is T put on the heap together with an atomic refcount

## Threads

- most types implements Send trait
  - it means the ownership of a value of the type can be transferred to
    another thread
- most types implements Sync trait
  - it means a value of the type can be referenced by multiple threads
  - T is Sync iff &T is Send
- `Rc<T>` implements neither
  - when a `Rc<T>` is shared by two threads, the refcount can be messed up
    refcount can be messed up
- `Arc<T>` implements both
