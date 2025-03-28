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
  - REMINDERS
    - ownership
      - a value is bits in memory
      - each value has a owner
      - when the owner of a value goes out of scope, the value is dropped
        - the compiler always knows the scope/liveness of the owner statically
    - reference
      - a reference borrows a value from its owner
      - a reference acts as if it owns the value
        - the value owner cannot go out of scope and drop the value while it
          is borrowed
        - there can be multiple immutable references at the same time
          - because none of the references can mutate the value, it is as if
            every reference owns the value
        - there can only be a single mutable reference at any time
          - otherwise, the value could be mutated by another mutable reference
      - a reference is itself the owner of a value as well
        - the value is the address of the referred value
        - when a reference goes out of scope, the address is dropped
    - lifetime of a reference
      - a reference owns a value (pointer) and borrows from another value
        (pointee)
      - when a reference goes out of scope, the pointer is dropped
        - iow, the scope/liveness of the reference ends
      - but when a reference goes out of scope, the pointee can still be
        borrowed!
        - iow, the scope/liveness of the reference ends, but the lifetime/loan
          of the reference may go on!
        - e.g., `fn foo<'a>(s: &'a str) -> &'a str { s }`
          - `s` is local to `foo` and goes out of scope when the fn ends
          - but the lifetime/loan goes on with the return value
      - iow, the scope/liveness of a reference and the lifetime/loan of a
        reference are separated
  - every reference has a lifetime
    - it is the scope for which a reference is valid
      - this is the confusing part
      - the scope/liveness of a reference (or any variable) is always known
        statically
      - the lifetime/loan of a reference is a separate concept
        - it is usually tied to the scope/liveness of the reference, but not
          always
    - the compiler can infer the lifetime usually
    - when it can't, the lifetime needs to be annotated
  - lifetime annotation syntax: `&'a i32`, `&'a mut i32`
    - explicit annotation does not change the liftime of any reference, just
      describes relations between references
      - this is also confusing
      - indeed explicit annotation merely gives a lifetime a name
      - but when two references are annotated with the same name, one of them
        has its lifetime changed (unless they already have the same lifetime
        to begin with)
    - annotate a single reference by itself is pointless
      - compiler knows the scope/liveness of the reference statically, and can
        tie the lifetime/loan to the scope/liveness
      - annotation is meant to describe relations between multiple references
  - lifetime annotation in function signatures: `fn foo<'a>`
    - the syntax of a generic lifetime is similar to that of a generic type
    - a note on subtyping
      - when `'b` outlives `'a`, `'b` is a subtype
      - that means a reference with `'b` can be used in places where `'a` is
        expected
      - that is different from generic type `T`, which must match exactly
    - the book emphasizes that annotations do not change lifetimes again
      - I think it is utterly wrong, or at least confusing
    - `fn longest<'a>(x: &'a str, y: &'a str) -> &'a str`
      - the book says that the compiler picks `'a` to be the overlap of `x`
        and `y`, and makes `'a` the lifetime of the return value
        - it reads to me that, since `x` and `y` are local to `longest`,
          compiler picks `'a` to be the scope of `longest` and the return
          value is useless
      - but I think what really happens is
        - compiler knows the scope/liveness of `x`, `y`, and the owner of the
          return value
          - e.g., `x` and `y` are local and are scoped to `longest`
        - compiler picks `'a` to be the scope/liveness of the owner of the
          return value
        - compiler requires `x` and `y` to outlive `'a`
        - as such, the caller keeps the inputs borrowed for as long as the
          return value is valid
    - e.g., `fn foo<'a>(x: &'a i32, y: &i32) -> &'a i32` means, as long as the
      returned reference is valid, so is `x`
    - e.g., `fn foo<'a>(x: &'a i32, y: &'a i32) -> &'a i32` means, as long as
      the returned reference is valid, so are `x` and `y`
  - lifetime annotation in structs: `struct Foo<'a>`
    - again, the syntax is the same as that of generic types
    - fields annotated with `'a` must outlive `'a`
      - in layman terms, when the struct is valid, those fields are also valid
    - the compiler does not try to infer the lifetime of the struct and
      requires explicit annotation
    - e.g., `struct Foo<'a>(&'a i32)` means, as long as the the struct is
      valid, so is the field
    - e.g., `struct Foo<'a>(&'a i32, &'a i32)` means, as long as the struct is
      valid, so are both fields
    - e.g., `struct Foo<'a, 'b>(&'a i32, &'b i32)` means...?
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
    - if there is no parameter that are references, the compiler does not infer
      the lifetime and requires explicit annotations
      - but why?
  - lifetime annotation on impls: `impl<'a>`
    - again, the syntax is the same as that of generic types
  - `'static`
    - all string literals have `'static` lifetime
      - they are stored in the binary directly
- e.g., `fn foo(s: &str) -> &str { s }` and `let a = String::new(); let b = foo(&a);`
  - we know that
    - `&a` creates a reference to `a` (`a` has one borrower, `&a`)
    - `s` is a copy of the reference (`a` has two borrowers, `&a` and `s`)
    - `foo` copies `s` to the return value (`a` has three borrowers, `&a`,
      `s`, and ret)
    - `s` goes out of scope but its lifetime goes on (`a` still has three
      borrowers)
      - its lifetime is extended by the inferred `'a`, which is tied to ret
    - `b` binds to the return value (`a` still has three borrowers)
    - `&a` goes out of scope  (`a` still has three borrowers)
    - at this point, only `b` is in scope and `a` is borrowed for as long as
      `b` is in scope
  - the compiler only sees the function signature, and assuming no lifetime
    inference, only knows that
    - `foo` takes a reference and returns a reference
    - `b` binds to the return value
    - it does not see the function body and does not know that `b` references
      `a`
  - with explicit annotation (or with lifetime inference), the signature
    becomes `fn foo<'a>(s: &'a str) -> &'a str`
    - the annotation describes the relation between two references
      - `s` and ret have the same lifetime
    - with that,
      - the compiler already always knows the scope of `b` statically
      - it ties `'a` to the scope of `b`
      - it knows `s` has the same lifetime `'a`
      - it knows `&a` has the same lifetime `'a`
      - as a result, `a` is borrowed for as long as `b` is in scope
    - note that there are cases when the compiler cannot infer the lifetimes
      and explicit annotation is required
      - e.g., `fn bar<s1: &str, s2: &str) -> &str`
- e.g., `fn foo<'a>(v: &'a i32) -> &'a i32 { body }`
  - body can be `v`, because `v` has `'a`
  - body can be `&3`, because `&3` has `'static` and outlives `'a`
    - this is different from generics
    - in `fn foo<T>(v: T) -> T`, the input and ret must have the same type
    - in this example, thanks to subtyping, ret only needs to outlive the
      input
- e.g., `fn foo<'a>() { body }`
  - this is pointless because `'a` is meant to describe relations between
    multiple references
  - when body is `let x = 3; let y: &'a i32 = &x;`, the compiler complains
    - the scope of `x` is the function body
    - it looks like `'a` is the scope of the call, which covers both the
      function body and the (non-existing) return value
    - that's why the compiler complains
- e.g., `let mut s = String::new();`
  - `s.is_empty()` is the same as `String::is_empty(&s)`
    - `&s` is a temp reference of `s` with the lifetime scoped to `is_empty`
    - `self` is a copy of `&s` (because all `&T` implements `Copy`)
  - `s.clear()` is the same as `String::clear(&mut s)`
    - `&mut s` is a temp reference of `s` with the lifetime scoped to `clear`
    - `self` is a move from `&mut s` (because `&mut T` does not implement `Copy`)
  - `let t = &s; t.is_empty();` is the same as `String::is_empty(t)`
    - `self` is a copy of `t`
  - `let t = &mut s; t.clear();` is the same as `String::clear(&mut *t)`
    - note that it is not the same as `String::clear(t)`
      - if it was, `t` would be moved because `&mut T` does not implement
        `Copy` and that would be undesirable
      - instead, the compiler rewrites `t` to `&mut *t` to "re-borrow" `t`
        - the compiler allows a mutable reference, or a subset of it, to be
          reborrowed
        - e.g., `let mut a = (1, 2); let b = &mut a; let c = &mut b.0;`
    - `&mut *t` is a temp reference of `t` with the lifetime scoped to `clear`
    - `self` is a move from `&mut *t`
- how I got confused
  - `&'a T`
    - I thought `'a` was both the lifetime and the scope of a reference
    - but the lifetime (or loan) and the scope (or liveness) of a reference
      are separated
      - lifetime only applies to references
      - scope applies to all variables, including references
    - `'a` only applies to the lifetime
    - compiler always knows the scope statically
  - `fn foo<'a, T>(v1: &'a T, v2: &'a T) -> &'a T`
    - I could not understand why `T` is an exact match but `'a` is not
    - but it is actually just subtyping
- `PhantomData`
  - this generates an error
    - `struct Foo<'a>(PhantomData<&'a()>);`
    - `fn foo<T>(_val: &T) -> Foo { Foo(PhantomData) }`
    - `fn main() { let r = { let v = 1; foo(&v) }; }`
  - `foo` has the signature `fn foo<'a, T>(_val: &'a T) -> Foo<'a>`
  - `v` has the lifetime of the inner block in `main`
  - `foo(&v)` has the same lifetime as `v` and cannot be assigned to `r` in
    the outer block

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
