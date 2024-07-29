The Rust Programming Language
=============================

## Chapter 1. Getting Started

- 1.1. Installation
- 1.2. Hello, World!
- 1.3. Hello, Cargo!

## Chapter 2. Programming a Guessing Game

## Chapter 3. Common Programming Concepts

- 3.1. Variables and Mutability
- 3.2. Data Types
- 3.3. Functions
- 3.4. Comments
- 3.5. Control Flow

## Chapter 4. Understanding Ownership

- 4.1. What is Ownership?
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
- 4.2. References and Borrowing
  - References for a variable can be created.  They "borrow" the onwership from
    the variable.
    - there are also "unsafe" pointers for a variable
  - a variable's ownership of a value can be immutably borrowed
    - `let a = 3; let b = &a; let c = &a;`
    - there can be multiple immutable borrowers
    - until `b` and `c` go out of scope, a's ownership is immutably borrowed.
      `a` can still be used, but its ownership of the value cannot be
      transferred
  - a variable's ownership of a value can also be mutably borrowed
    - `let mut a = 3; let b = &mut a;`
    - there can only be one mutable borrower
    - until `b` goes out of scope, `a`'s ownership is mutably borrowed.  `a`
      cannot be used.
  - a variable cannot be borrowed mutably and immutably at the same time
  - Internally, a reference is a pointer to the original value.  The borrowing
    rules make sure there is no surprise
- 4.3. The Slice Type
  - arrays have type `[T; N]`; different Ns mean different types
  - slices have type `[T]`
    - slices have unknown size.  Similar to `str` or `Path`, there can only be
      references to slices
    - a slice borrows from an arayy (or some collections)
  - vectors implements trait `Deref<Target = [T]>`, meaning they can be treated
    like slices
    - looping vectors in for-loops moves because of `into_iter`
    - looping slices in for-loops copies because references implement Copy trait
  - Internally, a slice is two pointers, delimiting a region in the original
    value.  The borrowing rules maek sure there is no surprise.
    - this is why slices are for types that are contiguous in memory

## Chapter 5. Using Structs to Structure Related Data

- 5.1. Defining and Instantiating Structs
  - `struct Foo {...}` defines a struct
- 5.2. An Example Program Using Structs
- 5.3. Method Syntax
  - `impl Foo {...}` defines associated funcs for `Foo`
  - an associated func is a method only when its first param is some variant
    of `self`
    - `&self` is replaced by `self: &Self`
    - `Self` is replaced by `Foo`, because we are in `impl Foo` block
  - e.g., in `let mut v = Vec::new(); v.push(1);`,
    - `new` is not a method
    - `push` is a method

## Chapter 6. Enums and Pattern Matching

- 6.1. Defining an Enum
  - `Option` enum
    - enum vals are `None` and `Some(T)`
- 6.2. The match Control Flow Construct
- 6.3. Concise Control Flow with if let

## Chapter 7. Managing Growing Projects with Packages, Crates, and Modules

- rust module system
  - package: a cargo feature to build, test, and share crates
  - crates: a tree of modules that produce a library or executable
  - modules and uses: code organization and scoping
  - paths: a way of naming items such as structs, functions, and modules
- 7.1. Packages and Crates
  - a package is a bundle of crates
  - `Cargo.toml` describes how to build the crates
  - there is at most one library crate
    - by convention, the crate root is `src/lib.rs`
  - there can many binary crates
    - by convention, the crate root is `src/main.rs`
    - each file under `src/bin/` is a separate binary crate
- 7.2. Defining Modules to Control Scope and Privacy
  - modules are used to organize and scope code
  - code within a module is private by default
  - single-file module tree
    - `src/lib.rs` or `src/main.rs` is itself a module named `crate`
    - `mod foo { mod bar {...} ... }` in the file defines
      - module `crate::foo`
      - module `crate::foo::bar`
  - multi-file module tree
    - cargo only builds `src/lib.rs` or `src/main.rs`
    - `pub mod foo;` in the file tells rustc to build `src/foo.rs`
    - `pub mod bar;` in `src/foo.rs` tells rustc to build `src/foo/bar.rs`
  - there are also `pub(crate)`, `pub(super)`, etc., to export an item to the
    current crate or parent crate
- 7.3. Paths for Referring to an Item in the Module Tree
  - `pub` marks a `mod`/`fn`/`struct`/`enum` public
  - `crate::` refers to the root module
  - `super::` refers to the parent module
- 7.4. Bringing Paths Into Scope with the use Keyword
  - `use` brings a name into scope
    - `use ... as ...` to pick an alternative name
  - idiomatically,
    - if `crate::foo::bar` is a fn,
      - `use crate::foo;` and refer to it as `foo::bar`
    - if `crate::foo::Baz` is a struct,
      - `use crate::foo::Baz;` and refer to it as `Baz`
  - `pub use ...` makes a name public
  - external packages
    - if `Cargo.toml` lists external package `foo` as a dependency,
      - `foo` refers to the root module
  - `use std::io::{self, Write};` expands to
    - `use std::io;`
    - `use std::io::Write;`
  - `use foo::*` brings all public items under `foo` into scope
- 7.5. Separating Modules into Different Files
  - we can build a module tree in a single file
  - we can also build a module tree using multiple files and directories
  - cargo knows `src/main.rs` or `src/lib.rs` is the crate root
    - `mod foo` in the crate root tells cargo to build `src/foo.rs`
      - or the legacy `src/foo/mod.rs`
    - `mod bar` in `src/foo.rs` tells cargo to build `src/foo/bar.rs`
      - or the legacy `src/foo/bar/mod.rs`

## Chapter 8. Common Collections

- 8.1. Storing Lists of Values with Vectors
- 8.2. Storing UTF-8 Encoded Text with Strings
  - ways to convert `&str` to `String`
    - `"test".to_string()`, because `str` implements the
      `std::string::ToString` trait
      - the trait is automatically implemented for any type that implements
        the `Display` trait
    - `"test".to_owned()`, because `str` implements the `std::borrow::ToOwned`
      trait
      - `str` implemenets `ToOwned` with
        `String::from_utf8_unchecked(self.as_bytes().to_owned())`
    - `String::from("test")` because `String` implements `From<&str>`
      - it implements `From<&str>` with `s.to_owned()`
- 8.3. Storing Keys with Associated Values in Hash Maps

## Chapter 9. Error Handling

- 9.1. Unrecoverable Errors with `panic!`
- 9.2. Recoverable Errors with `Result`
  - `Result` enum
    - enum vals are `Ok(T)` and `Err(E)`
  - panic on errors
    - `unwrap` and `expect`
  - early return on errors
    - `?` operator
      - if `Ok(T)`, unwraps and returns `T`
      - if `Err(E)`, early returns `Err(E)`
        - it actually early returns `Err(From::from(e))`, to cast the error
          type when necessary
      - it works with `Option` as well
  - `std::error::Error` trait
    - while there is no restriction on `Err(E)`, it is generally a good idea
      for `E` to implement the `Error` trait
    - the `Error` trait requires `Debug` and `Display` traits
      - `pub trait Error: Debug + Display`
      - that is, a type implementing `Error` must also implement `Debug` and
        `Display`
    - the `Error` trait has these methods
      - `description` is deprecated because the `Display` trait req serves the
        same purpose
      - `cause` is deprecated by `source`
      - `source` returns `Option<&dyn Error>`, which is an optional low-level
        error wrapped in this high-level error (that is, the root source of
        this error)
      - `provide` returns associated payload (often backtrace) with this error
      - iow, when a user chooses to deal with the generic `Error` trait rather
        than each concrete error type, the trait provides
        - `Display` trait to return a string that describes this error
        - `source` method to return the root cause, if any
        - `provide` method to return additional info, if any
  - `std::io::Error` implements `std::error::Error` trait
    - `io::Error::new` creates an error from `ErrorKind` and a payload
      - `std::io::ErrorKind` is a plan enum
      - the payload can be many things, such as a string or another error
    - `io::Error::from_raw_os_error` creates an error from
      `std::io::RawOsError`, which is the os errno
  - `std::io::Result` is an alias of `std::result::Result<T, std::io::Error>`
  - best practices
    - our code generates `N` kinds of errors
    - our dependencies generate `M` error types
    - if we want to faithfully return the errors to the user, define a custom
      error type
      - an enum with `N+M` vals, with the `M` vals wrapping the `M` error
        types
      - impl `std::error::Error` to allow the user to handle the custom error
        generically
      - impl `From<T>` to allow the user (and ourselves) to propagate from
        `Err(T)` to our custom error type using `?`
      - this is a common case for libraries and `thiserror` crate is designed
        for the case
    - if we don't expect users to care and we just want to log the errors
      somewhere, define a trait object for `std::error::Error`
      - the trait object gives us a string representation of the error (and
        a bit more)
      - this is a common case for executables and `anyhow` create is designed
        for the case
- 9.3. To `panic!` or Not to `panic!`

## Chapter 10. Generic Types, Traits, and Lifetimes

- 10.1. Generic Data Types
  - turbofish syntax
    - when we have `struct Foo<T>`, we use `Foo<i32>` to specify a concrete
      type
    - when we have `fn foo<T>`, we use `foo::<i32>` instead to specify a
      concrete type
      - `::<>` is nicknamed turbofish
    - when `struct Foo<T>` has method `foo`, we use `Foo::<i32>::foo`
- 10.2. Traits: Defining Shared Behavior
  - `trait Foo { fn bar(&self)... }` defines trait `Foo`
  - `impl Foo for Baz` implements trait `Foo` for type `Baz`
    - trait `Foo` can also have non-methods such as `fn blah()`
      - that is, no `self`
      - `Baz::blah` invokes the func
      - `Foo::blah` also invokes the func if the type can be inferred
      - e.g.,
        - `let a : i32 = Default::default();`
        - `let b = i32::default();`
        - `let c = <i32 as Default>::default();`
  - trait bound
    - `fn some_func(foo: &impl Foo)` takes any type that implements
      `Foo` trait
    - it is a syntax sugar for `fn some_func<T: Foo>(foo: &T)`
      - or the newer `fn some_func<T>(foo: &T) where T: Foo`
    - multiple trait bounds are possible with `+`
      - `fn some_func(foo: &(impl Foo + Bar))`
      - `fn some_func<T: Foo + Bar>(foo: &T)`
    - `Sized` trait is special
      - if a type has a fixed-size at compile-time (not DST), the compiler
        implements `Sized` for the type automatically
      - `T` is assumed to have `Sized` bound (except for `Self` in traits)
      - `T: !Sized` means no `Sized` bound
      - `T: ?Sized` means either `Sized` or not
  - `trait Foo: Bar` means a type must implement `Bar` first to implement `Foo`
  - trait coherence
    - the goal is to make sure the compiler can always unambiguously resolve a
      trait func call to an impl
    - the overlay rule
      - if a type implements two traits, there must be no overlap between the
        two traits
    - the orphan rule
      - if a type implements a trait, at least one of them must be defined by
        the current crate
- 10.3. Validating References with Lifetimes
  - every reference has a lifetime
    - it is the scope for which a reference is valid
    - the compiler can infer the lifetime usually
    - when it can't, the lifetime needs to be annotated
  - lifetime annotation syntax
    - on references: `&'a i32`, `&'a mut i32`
    - on functions: `fn foo<'a>`
    - on structs: `struct Foo<'a>`
    - on impls: `impl<'a>`
  - when a function returns a reference, the reference has a lifetime
    - the compiler knows that it either refers to a global variable or to a
      function parameter that also is a reference
      - it can't refer to owned parameters or local variables, because those
        die when the function returns
    - if there is only one parameter that is a reference, the compiler infers
      the lifetime of the return value to be the lifetime of the parameter
      - this is always safe, although when the return value refers to a global
        variable, the actual lifetime can be longer (`'static`)
    - if there are multiple parameters that are references, the compiler can't
      infer the lifetime and requires explicit annotations
    - if there is no parameter that are references, the comper does not infer
      the lifetime and requires explicit annotations
      - but why?
    - `fn foo<'a>(x: &'a i32, y: &i32) -> &'a i32` means the returned
      reference is valid when `x` is
    - `fn foo<'a>(x: &'a i32, y: &'a i32) -> &'a i32` means the returned
      reference is valid when both `x` and `y` are
  - when a struct has fields that are references, the struct has a lifetime
    - the compiler does not try to infer the lifetime of the struct and
      requires explicit annotation
    - `struct Foo<'a>(&'a i32)` means the struct is valid when the field is
    - `struct Foo<'a>(&'a i32, &'a i32)` means the struct is valid when both
      fields are
    - `struct Foo<'a, 'b>(&'a i32, &'b i32)` means...?
  - lifetime elision
    - the compiler assigns lifetimes following 3 rules
      - it assigns lifetimes to all input references
      - if there is only one input reference, its lifetime is assigned to all
        output references
      - if there are multiple input references, and one of them is `&self` or
        `&mut self`, its lifetime is assigned to all output references
        - this sounds incorrect but is true
        - the compiler generates an error when the method does not return self
          or a field of self
    - unless all output references have assigned lifetimes, the compiler
      generates an error and requires explicit annotations
  - all string literals have `'static` lifetime
    - they are stored in the binary directly
    - do numeric literals also have `'static` lifetime?
- I still don't get it.  Let's restart.
  - value, type, and ownership
    - a value is stored in memory
      - it can be stored in stack, heap, or `'static`
      - to access a value, we must know its addr and size
    - a value must have a type
      - the type decides the interpretation
      - unless the type is DST, it also decides the size
    - a value must have an owner
      - when it loses the owner, it is dropped
      - the ownership can be moved to a different owner
    - a value can have borrowers
      - there are several rules, originated from the idea that each borrower
        acts as if it owns the value despite it does not
      - when there is any borrower, the owner cannot mutate the value
        - because each borrower thinks it is the owner, it does not expect
          the real owner to mutate the value behind its back
      - there can be multiple immutable borrowers
        - each immutable borrower as well as the owner sees the immutable
          value
      - there can be at most one mutable borrower
        - the mutable borrower gains full access to the value
        - the owner loses access to the value
  - pattern and variable
    - a pattern can match a value, optionally binds the value to a variable
      - in `let a = 3`, `a` is a pattern, `3` is a value, and `a` binds to `3`
      - in `let mut a = 3; a = a + 1;`
        - `a` matches and binds to `3`
        - `a + 1` evaluates to a temp
        - `a` matches and binds to the temp
    - the variable becomes the owner of the value, or a copy of the value when
      its type supports `Copy` trait
      - in `let a = 3; let b = &a; let c = b;`
        - `a` is the owner of `3`
        - `&a` evalutes to a temp
        - `b` binds to a copy of the temp, because `&T` implements the `Copy`
          trait
        - `c` binds to a copy of `b`
    - when the variable goes out of scope, the value loses its owner and is
      dropped
  - lifetime
    - remember this is done by the borrow checker during static analysis
      - any use of an invalid variable causes a compile-time error
    - a value has a lifetime
      - starts when the value is allocated/initialized
      - ends when the value is dropped
    - when a variable is the owner of a value, the variable is valid until the
      value gets moved out
      - the value gets a new owner and the current variable becomes invalid
    - when a variable is a borrower of a value, the variable is valid until
      the the borrowed value is dropped
      - if the borrowed value has type `T` and lifetime `'a`, the variable
        binds to a value of type `&'a T`
        - the variable, or the value it binds to, is called a reference
      - the borrow has a lifetime too
        - starts when the reference is allocated/initialized
        - ends after the reference is last used
      - if the lifetime of the borrow is not a subset of `'a`, the borrow
        checker generates a compile-time error
      - e.g., `let a; {a = &vec!(1)}; a;` generates an error that goes away
        after removing the last statement (to shorten the lifetime of the
        borrow) or removing the curly braces (to lengthen the lifetime of the
        value)
- experiments with function lifetime annotations
  - `fn foo<'a>()`
    - `let x: &'static i32 = &3;` passes
      - it claims the referenced value outlives `'static`, which is correct
        because `3` has the lifetime of `'static`
    - `let x: &'a i32 = &3;` passes
      - it claims the referenced value outlives `'a`, which is correct because
        `3` has the lifetime of `'static`
    - `let x = 3; let y: &'static i32 = &x;` fails
      - it claims the referenced value outlives `'static`, which is incorrect
        because `x` has the lifetime of the body
    - `let x = 3; let y: &'a i32 = &x;` fails
      - it looks like `x` does not outlive `'a`, suggesting that `'a` strictly
        outlives the body
  - `fn foo<'a>() -> &'a i32`
    - `&3` passes, because `3` is `'static` and outlives `'a`
    - `const C: i32 = 3; &C` passes, because `C` is `'static` too
    - `let x = 3; &x` fails, suggesting `'a` strictly outlives the body
  - `fn foo<'a>(v: &'a i32)`
    - `let x: &'a i32 = v;` passes, because `*v` is `'a`
    - `let x: &'a i32 = &3;` passes, because `3` is `'static` and outlives
      `'a`
      - this is known as subtyping: when `'b` outlives `'a`, `&'b T` is a
        subtype of `&'a T` and we can treat `&'b T` as `&'a T`
    - `let x = 3; let mut y = &x; y = &3; y = v;` passes
      - when annotated with `'_` or not annotated, `&'b i32` is a subtype for
        any `'b` and `y` can bind to any of them
    - `let x = 3; let y: &'a i32 = &x;` fails, suggesting that `'a` strictly
      outlives the body
      - this makes sense because when a caller calls this function, the input
        decides `'a` and the input outlives the body
    - `let x: &'static i32 = v` fails, suggesting that `'a` does not always
      outlive `'static`
      - we can add a lifetime bound `'a: 'static` to make it pass, but it
        also restricts what the function takes
      - this is similar to generics and trait bounds
  - `fn foo<'a>(v: &'a i32) -> &'a i32`
    - `v` passes, because `*v` is `'a`
    - `&3` passes, because `3` is `'static` and outlives `'a`
      - this is different from generics
      - in `fn foo<T>(v: T) -> T`, the input and ret must have the same type
      - in this example, thanks to subtyping, ret only needs to outlive the
        input
    - what happens internally is probably
      - the compiler sees a call to the function
      - the input references a value of concrete lifetime `'x`
      - the output type has an annotated lifetime `'y`
        - it is normally not annotated which implies `'_`
      - the compiler replaces `'a` by `'x`
      - the compiler generates an error if `'x` does not outlive `'y`
        - thanks to subtyping, `'x` and `'y` does not need to match
  - `fn foo<'a>(v1: &'a i32, v2: &'a i32) -> &'a i32`
    - what happens internally is probably
      - the compiler sees a call to the function
      - the inputs reference a value of concrete lifetime `'x` and another
        value of concrete lifetime `'y`
      - the output type has an annotated lifetime `'z`
        - it is normally not annotated which implies `'_`
      - the compiler replaces `'a` by `intersection('x, 'y)`
      - the compiler generates an error if `intersection('x, 'y)` does not
        outlive `'z`
        - thanks to subtyping, `'x`, `'y`, and `'z` does not need to match
- how I got confused
  - `&'a T`
    - I thought `'a` was the lifetime of the borrow
    - but it is actually the lifetime of the referenced value
  - `fn foo<'a, T>(v1: &'a T, v2: &'a T) -> &'a T`
    - I could not understand why `T` is an exact match but `'a` is not
    - but it is actually just subtyping

## Chapter 11. Writing Automated Tests

- 11.1. How to Write Tests
  - `cargo test` buids a test runner to run functions with `#[test]` attr
- 11.2. Controlling How Tests Are Run
  - `cargo test --help` for help
  - `cargo test -- --help` for test harness help
    - `--nocapture` or `--show-output` to show output
- 11.3. Test Organization
  - unit tests are added to modules directly
    - in `mod tests` submodules decorated with `#[cfg(test)]`
  - integration tests are added to `tests/`

## Chapter 12. An I/O Project: Building a Command Line Program

- 12.1. Accepting Command Line Arguments
- 12.2. Reading a File
- 12.3. Refactoring to Improve Modularity and Error Handling
- 12.4. Developing the Library’s Functionality with Test Driven Development
- 12.5. Working with Environment Variables
- 12.6. Writing Error Messages to Standard Error Instead of Standard Output

## Chapter 13. Functional Language Features: Iterators and Closures

- 13.1. Closures: Anonymous Functions that Capture Their Environment
- 13.2. Processing a Series of Items with Iterators
- 13.3. Improving Our I/O Project
- 13.4. Comparing Performance: Loops vs. Iterators

## Chapter 14. More about Cargo and Crates.io

- 14.1. Customizing Builds with Release Profiles
- 14.2. Publishing a Crate to Crates.io
- 14.3. Cargo Workspaces
- 14.4. Installing Binaries from Crates.io with cargo install
- 14.5. Extending Cargo with Custom Commands

## Chapter 15. Smart Pointers

- 15.1. Using `Box<T>` to Point to Data on the Heap
  - `let a = 5` and `let b = Box::new(5)`, one is on stack and the other is on
    heap
  - `Box<T>` is commonly used when
    - `T` is a recursive type, because rustc needs to know the type size at
      compile time
    - `T` is huge and we want to make sure it is never copied
    - `T` is a trait object
  - `Box<T>` implements `Deref` and `Drop` traits
- 15.2. Treating Smart Pointers Like Regular References with the `Deref` Trait
  - with `let x = 5;` and `let y = &x;`, `y` is a reference to `x` and must be
    dereferenced to get the value, such as in `*y == x`
  - if we have `let y = Box::new(x);` instead, we can still do `*y == x`
    because `Box` implements `Deref` trait
    - it implements the `deref` method which takes `&Box<T>` and returns `&T`.
    - that allows the compler to replace `*y` by `*(y.deref())`
  - deref coercion
    - it is for convenience
    - when a function takes `&str`, we can pass `&Box<String>` to it thanks to
      deref coercion
    - `&Box<String>` becomes `&String` which becomes `&str` through a series
      of `deref()`
  - there is also `DerefMut` trait for `&mut`
  - `Deref` and `AsRef`
    - when `T` wraps `U`, implementing `Deref` for `T` enables type coercion
      (implicit type conversion) to `U`
      - this is very convenient, and the key consideration is if the implicit
        conversion can be an unexpected surprise or not
      - this is done for `Box<T>` to `T`, `String` to `str`, etc.
    - when `T` casts to `U` at little or no cost, implementing `AsRef<U>` for
      `T` enables generic functions
      - e.g., if I have a function that takes a path, the param can be generic
        with trait bound `AsRef<Path>` to accept a wide range of types, such
        as `str`, `String`, `OsStr`, `OsString`, `Path`, `PathBuf`, etc.
- 15.3. Running Code on Cleanup with the `Drop` Trait
  - `Box` implements the `Drop` trait
    - when it goes out of scope, `drop` method is called automatically to free
      the storage
  - to drop early, `std::mem::drop` can be used
    - the prelude has `use std::mem::drop;` so it is just `drop`
- 15.4. `Rc<T>`, the Reference Counted Smart Pointer
  - `Rc<T>` is T put on the heap together with a refcount
    - like `std::shared_ptr<T>`
  - `Arc<T>` is T put on the heap together with an atomic refcount
- 15.5. `RefCell<T>` and the Interior Mutability Pattern
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
- 15.6. Reference Cycles Can Leak Memory

## Chapter 16. Fearless Concurrency

- 16.1. Using Threads to Run Code Simultaneously
- 16.2. Using Message Passing to Transfer Data Between Threads
- 16.3. Shared-State Concurrency
  - most types implements `Send` trait
    - it means the ownership of a value of the type can be transferred to
      another thread
  - most types implements `Sync` trait
    - it means a value of the type can be referenced by multiple threads
    - `T` is `Sync` iff `&T` is `Send`
  - `Rc<T>` implements neither
    - when a `Rc<T>` is shared by two threads, the refcount can be messed up
  - `Arc<T>` implements both
- 16.4. Extensible Concurrency with the Sync and Send Traits

## Chapter 17. Object Oriented Programming Features of Rust

- 17.1. Characteristics of Object-Oriented Languages
- 17.2. Using Trait Objects That Allow for Values of Different Types
  - given `trait Foo {}` and `impl Foo for Bar`
    - `Bar` implements the `Foo` trait
    - when a method is called, it can be resolved statically and no vtable is
      needed
  - on the other hand, `Box<dyn Foo>` is a trait object
    - `dyn` makes sure there is a vtable
    - when a method is called, it is resolved dynamitcally via the vtable
    - note that `Box<dyn Foo>` is a fat pointer
      - `std::mem::size_of::<Box<dyn Foo>>()` returns 16
      - there is 1 pointer to the concrete type and 1 pointer to the vtable
- 17.3. Implementing an Object-Oriented Design Pattern

## Chapter 18. Patterns and Matching

- 18.1. All the Places Patterns Can Be Used
- 18.2. Refutability: Whether a Pattern Might Fail to Match
- 18.3. Pattern Syntax

## Chapter 19. Advanced Features

- 19.1. Unsafe Rust
- 19.2. Advanced Traits
- 19.3. Advanced Types
- 19.4. Advanced Functions and Closures
- 19.5. Macros

## Chapter 20. Final Project: Building a Multithreaded Web Server

- 20.1. Building a Single-Threaded Web Server
- 20.2. Turning Our Single-Threaded Server into a Multithreaded Server
- 20.3. Graceful Shutdown and Cleanup

## Chapter 21. Appendix

- 21.1. A - Keywords
- 21.2. B - Operators and Symbols
- 21.3. C - Derivable Traits
  - `#[derive]` attr on a struct or an enum generates a default implementation
    for the specified trait
  - `#[derive(Debug)]` enables string formatting with `{:?}`
  - `#[derive(PartialEq)]` and `#[derive(Eq)]` enables `==` and `!=`
    - `Eq` trait cannot be implemented on floats, because `NaN != NaN`
  - `#[derive(PartialOrd)]` and `#[derive(Ord)]` enables `<`, `>`, `<=`, and
    `>=`
    - `Eq` trait cannot be implemented on floats, because `NaN` has no
      ordering
  - `#[derive(Clone)]` and `#[derive(Copy)]`
    - `Copy` trait is fast because it copies bits on stack
    - `Clone` trait can be slow becaise it might copies bits on heap
  - `#[derive(Hash)]` eanbles hashing
  - `#[derive(Default)]` enables default value
- 21.4. D - Useful Development Tools
- 21.5. E - Editions
- 21.6. F - Translations of the Book
- 21.7. G - How Rust is Made and “Nightly Rust”
