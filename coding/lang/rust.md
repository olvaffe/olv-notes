Rust
====

## Resources

- <https://github.com/rust-lang>
  - <https://github.com/rust-lang/rust> has rustc, std, and docs
    - <https://github.com/rust-lang/backtrace-rs> is `std::backtrace`
    - <https://github.com/rust-lang/stdarch> is `std::arch`
    - <https://github.com/rust-lang/book>
    - <https://github.com/rust-lang/edition-guide>
    - <https://github.com/rust-embedded/book>
    - <https://github.com/rust-lang/nomicon> is advanced/unsafe rust
    - <https://github.com/rust-lang/reference>
    - <https://github.com/rust-lang/rust-by-example>
    - <https://github.com/rust-lang/rust/tree/master/src/doc/style-guide>
    - <https://github.com/rust-lang/rust/tree/master/src/doc/unstable-book>
  - binaries
    - <https://github.com/rust-lang/cargo>
    - <https://github.com/rust-lang/rust-bindgen>
    - <https://github.com/rust-lang/rust-clippy>
    - <https://github.com/rust-lang/rustfmt>
    - <https://github.com/rust-lang/rustup>
  - libraries
    - <https://github.com/rust-lang/cfg-if>
    - <https://github.com/rust-lang/flate2-rs>
    - <https://github.com/rust-lang/futures-rs>
    - <https://github.com/rust-lang/getopts>
    - <https://github.com/rust-lang/glob>
    - <https://github.com/rust-lang/libc>
    - <https://github.com/rust-lang/log>
    - <https://github.com/rust-lang/regex>
    - <https://github.com/rust-lang/socket2>
  - docs
    - <https://github.com/rust-lang/api-guidelines>
    - <https://github.com/rust-lang/async-book>
    - <https://github.com/rust-lang/rfcs>
- <https://github.com/rust-unofficial>
  - <https://github.com/rust-unofficial/awesome-rust>
  - <https://github.com/rust-unofficial/patterns>
- popular crates
  - <https://crates.io/crates/anyhow>
  - <https://crates.io/crates/bitflags>
  - <https://crates.io/crates/bytes>
  - <https://crates.io/crates/cfg-if>
  - <https://crates.io/crates/clap>
  - <https://crates.io/crates/either>
  - <https://crates.io/crates/itertools>
  - <https://crates.io/crates/libc>
  - <https://crates.io/crates/log>
  - <https://crates.io/crates/memoffset>
  - <https://crates.io/crates/once_cell>
  - <https://crates.io/crates/proc-macro2>
  - <https://crates.io/crates/quote>
  - <https://crates.io/crates/regex>
  - <https://crates.io/crates/scopeguard>
  - <https://crates.io/crates/serde>
  - <https://crates.io/crates/smallvec>
  - <https://crates.io/crates/syn>
  - <https://crates.io/crates/thiserror>

## Top GitHub Repos

- <https://github.com/denoland/deno>, node-like javascript runtime
- <https://github.com/tauri-apps/tauri>, html-based desktop gui framework
- <https://github.com/rustdesk/rustdesk>, remote desktop
- <https://github.com/alacritty/alacritty>, terminal
- <https://github.com/BurntSushi/ripgrep>, fast grep
- <https://github.com/dani-garcia/vaultwarden>, bitwarden server impl
- <https://github.com/bevyengine/bevy>, game engine
- <https://github.com/astral-sh/ruff>, python formatter
- <https://github.com/swc-project/swc>, transpile modern js to legacy js
- <https://github.com/tokio-rs/tokio>, async io
- <https://github.com/vercel/turborepo>, js bundler
- <https://github.com/iced-rs/iced>, cross-platform gui library
- <https://github.com/rwf2/Rocket>, (server-side) web framework
- <https://github.com/emilk/egui>, immediate-mode gui library
- <https://github.com/cloudflare/pingora>, (server-side) web framework
- <https://github.com/actix/actix-web>, (server-side) web framework
- <https://github.com/tokio-rs/axum>, (server-side) web framework
- <https://github.com/leptos-rs/leptos>, (client-side) web framework
- <https://github.com/hyperium/hyper>, http request
- <https://github.com/clap-rs/clap>, arg parser

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

- <https://rust-lang.github.io/rustup/cross-compilation.html>
- `rustup target list` for supported targets
- `rustup target add <triple>` to add the toolchain
- `cargo build --target <triple>` to build
- a linker must be specified separately
  - edit `~/.cargo/config.toml` to add

    [target.<triple>]
    linker = "..."
  - or, set the `CARGO_TARGET_<triple>_LINKER` envvar
- aarch64
  - `rustup target add aarch64-unknown-linux-gnu`
  - `cargo build --target aarch64-unknown-linux-gnu`
  - install the linker separately and edit `~/.cargo/config.toml`

    [target.aarch64-unknown-linux-gnu]
    linker = "aarch64-linux-gnu-gcc"
- android
  - `rustup target add aarch64-linux-android`
  - `cargo build --target aarch64-linux-android`
  - install the ndk and edit `~/.cargo/config.toml`

    [target.aarch64-unknown-linux-gnu]
    linker = "<ndk>/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android34-clang"

## cargo

- `~/.cargo` or `$CARGO_HOME`
- `$CARGO_HOME/registry`
  - `index` appears to be
    `git clone --bare https://github.com/rust-lang/crates.io-index.git`
  - `cache` contains downloaded `.crate` files (can be renamed to `.tar.gz`)
  - `src` contains un-tared crates
- `cargo new <dir>` creates a new project
  - `src/main.rs` is the source code
  - `Cargo.toml` is the manifest
- `cargo build`
  - downloads dependencies to `$CARGO_HOME/registry/src`
  - builds dependencies in `target/debug/deps`
  - `cargo build --examples` builds binary crates under `src/examples`
- `cargo test`
  - it builds and runs tests
  - unit tests go to `src/`
  - integration tests go to `tests/`
- `cargo bench`
  - it builds and runs benches
  - unit benches go to `src/`
  - integration benches go to `benches/`

## API Guidelines

- <https://rust-lang.github.io/api-guidelines/>
- Chapter 1. Naming
  - when camel casing, `Uuid`, `Usize`, `Stdin`, etc.
  - ad-hoc conversoins
    - `as_*` is free and is between refs
      - it conceptually returns a view of the underlying type
      - all string types have `as_*`
    - `to_*` has a cost and can fail
      - it conceptually converts from one type to another
      - `str`, `OsStr`, and `cstr` have `to_*`
      - `String`, `OsString`, and `CString` do not
    - `into_*` consumes self
      - it conceptually reuses the underlying storage
      - `String`, `OsString`, and `CString` have `into_*`
      - `str`, `OsStr`, and `cstr` do not (unless they are in `Box`)
      - if the type is a wrapper to another, `into_inner` returns the other
        type
  - getters do not have `get_` prefix
    - unless the type has only a single getter, then it is called `get`
  - a collection type should have `iter`, `iter_mut`, and `into_iter`
  - feature names are concise, `foo` instead of `with_foo`
- Chapter 2. Interoperability
  - implements all common traits that make sense
    - `Default`
      - if `Default` makes sense, `new` with no arg makes sense too
    - `Copy`, `Clone`
    - `Debug`, `Display`
    - `Eq`, `PartialEq`, `Ord`, `PartialOrd`
    - `Hash`
    - because of the orphan rule, a client cannot implement common traits for
      types defined by a crate
  - implements all conversion traits that make sense
    - `From`, `TryFrom`
      - never implements `Into` or `TryInto`, as they are implemented via
        `From` or `TryFrom`
      - `?` operator inserts `from` so this is especially useful for error
        types
    - `AsRef`, `AsMut`
      - e.g., a function that accepts a path should be
        `fn foo(path: impl AsRef<Path>)` to accept as many types as possible
  - implements `FromIterator` and `Extend` for collections
    - to enable `Iterator::collect`, `Iterator::partition`, etc.
  - implements `Serialize` and `Deserialize` from `serde` for types that hold
    pure data
  - implements `Send` and `Sync` whenever possible
  - implements `Error` for error types
    - also `Send` and `Sync`
    - also `Debug` and `Display`
  - implements `std::fmt::*` such as `std::fmt::LowerHex` for bitflag types
  - takes `T: Read` or `T: Write` params by values
    - it will work with `T` or `&mut T`, as std implements the trait for `&mut
      T`
- Chapter 3. Macros
- Chapter 4. Documentation
- Chapter 5. Predictability
  - no non-static inherent methods on smart pointers
    - `impl Foo { fn bar(p: Self) {} }` instead of
      `impl Foo { fn bar(self) {} }`
    - it enforces the use of `Foo::bar(p)` rather than `p.bar()`
    - it seems
      - `impl Foo { ... }` are inherent methods
        - if a method takes `self` as the first arg, it is non-static
      - `impl Bar for Foo { ... }` are trait methods
  - define conversion methods on the more specific type if there is one
    - e.g., we can convert between `str` and `[u8]` and `str` is more specific
      - conversion methods are defined for `str`, such as `str::as_bytes` or
        `std::str::from_utf8`
    - when defining methods, prefer `to_*`/`as_*`/`into_*` over `from_*`
      because `val.to_foo()` is more ergonomic than `Foo::from(val)`
  - when a function acts on a type, make it a method
  - prefer return vals over out params
    - prefer `fn foo() -> Out` over `fn foo(&mut Out)`
    - this does not apply when `foo` modifies the data than creating the data
  - no surprise with `std::op` traits
    - they are for operator overloading and should not have any surprise
    - implements `std::ops::Deref` only for smart pointers
      - because it participates in implicit type coercion and method
        resolution
      - that's how `&String` is coerced to `&str`, or how `val.foo()` works
        for both `&T` and `Box<T>`, transparently
  - no surprise with `std::op` traits
  - ctors
    - `new` takes no or obvious params
      - it usually does not fail, but it can
      - `String::new` and `OsString::new` take no param
      - `CString::new` and `OsStr::new` take a generic param
        - `CString::new` can fail
      - `str` and `CStr` have no `new`
    - `with_<variant>` (and `new_with_<variant>`) are variants of `new`
      - `String::with_capacity` and `OsString::with_capacity`
      - consider the builder pattern instead
    - `from_<another>` converts from another type
      - it usually takes ownership, but it may not
      - it is more flexible than the `From` trait
        - it can fail and it can take multiple params
      - `String`, `OsString`, `CString`, `OsStr`, and `CStr` have `from_*`
      - `str` does not
    - impl `Default` trait if makes sense
      - all string types implement the `Default` trait
      - it is often expected for both `new` and `Default` to exist and to
        behave the same
    - impl `From` trait if makes sense
      - it usually takes ownership, but it may not (e.g., `T` is a ref)
      - `String`, `OsString`, and `CString` implements `From`
      - `str`, `OsStr`, and `CStr` do not
- Chapter 6. Flexibility
  - if intermediate results of a function are interesting, return them as well
    - e.g., `Vec::binary_search` does not return just a bool or the idx
  - function param ownership
    - if a func requires ownership of a param, use `Foo`
      - do not clone in the func but let the caller decides
    - if a func does not require ownerhsip, use `&Foo` or `&mut Foo`
  - use generics to minimize assumptions on func params
    - e.g., `fn foo(path: impl AsRef<Path>)` rather than `fn foo(path: &Path)`
  - decide if a trait is to be used as a trait bound or a trait object or both
    - a trait is used as a trait bound in generics such as `T: Trait`
      - it means `T` is any type that implements `Trait`
    - a trait is used as a trait object types such as `dyn Trait`
      - `dyn Trait` is itself a type
      - it is a DST and cannot be instantiated
      - use `Box<dyn Trait>` instead
    - generics vs trait object
      - static dispatch vs dynamic dispatch
        - generics wins
      - regular pointer vs fat pointer
        - generics wins
      - homogeneous type vs heterogeneous types
        - generics is not an option...
    - trait object limits
      - trait objects are implemented via vtables
      - the compiler needs to generate vtables and all methods in the vtable
        must be fully known at compile-time
      - the traits cannot have generics at all
        - the trait itself can be generic, `trait Foo<T>`
        - but it cannot have generic methods, `trait Foo { fn bar<T>(t: T); }`
        - this is also allowed,
          - `trait Foo { type Bar; }` and `dyn Foo<Bar = Xxx>`
      - the traits also cannot have `Self` other than as the first param
        - because `Self` is like generics and its concrete type is unknown at
          compile-time
        - the first param is special because it is always a pointer to the
          object in the vtable
        - this can be worked around by excluding such methods from the vtable
          with `Self: Sized` bound
    - trait object examples
      - `std::io::Read` and `std::io::Write` are often used as trait objects
      - `std::iter::Iterator` can be used as trait objects
      - note how they all exclude some methods from vtables with `Self: Sized`
- Chapter 7. Type safety
  - newtype pattern
    - e.g., `struct Modifier(u64)` and uses `Modifier` for type safetyp
  - use enum or others in place of `bool` in func params
    - e.g., `sync(Start)` is better than `sync(true)`
  - use `bitflags` instead of `enum` for bitflags
  - builder pattern
    - non-consuming builders, such as `std::process::Command`
      - the config methods are `fn foo(&mut self...) -> &mut Self`
      - the terminal methods are `fn build(&self)`
    - consuming builders
      - the config methods are `fn foo(mut self...) -> Self`
      - the terminal methods are `fn build(self)`
- Chapter 8. Dependability
  - validate func args
    - static enforcement, such as the newtype pattern
    - dynamic enforcement, with runtime overhead
      - two common ways to avoid the overhead are
        - use `debug_assert!` to enable it for debug builds
        - add `foo_unchecked` variant or `raw` submodule
  - drop never fails
    - if cleanup can fail or can block, define a method for explicit cleanup
    - if the explicit cleanup is not called, drop should eat and log the
      failure
- Chapter 9. Debuggability
  - `Debug` should be implemented for all public types
  - `Debug` should output something even the val is empty, such as `""` for
    empty str
- Chapter 10. Future proofing
  - sealed traits to prevent clients from implementing the traits
  - keep struct fields private unless must
    - making a field public is a strong commitment
    - structs with private fields cannot be constructed with literals
  - return `impl Trait` if that's sufficient
    - e.g., return `impl Iterator<Item = u32>` allows clients to use the
      return value as an iterator while allowing us to change the concrete
      type at anytime
    - this can also be achieved with newtype pattern
  - do not duplicate derives and trait bounds for plain structs
    - `#[derive(Debug)] struct Foo<T>` is fine
      - adding another derive does not break clients
      - if `T` does not implement `Debug`, neither will Foo
    - `#[derive(Debug)] struct Foo<T: Debug>` is unnecessary and bad
      - adding another derive _and_ bound can break clients, because of the new
        bound requirement
- Chapter 11. Necessities
- api evolution
  - <https://rust-lang.github.io/rfcs/1105-api-evolution.html>

## Rust By Example

- <https://doc.rust-lang.org/stable/rust-by-example/>

## Rust Design Patterns

- <https://rust-unofficial.github.io/patterns/>

## Rustonomicon

- <https://doc.rust-lang.org/stable/nomicon/>

## The Rust Standard Library

- <https://doc.rust-lang.org/std/>
  - there is also <https://doc.rust-lang.org/core/>
    - core provides stuff that has no external dependency (such as libc,
      system libs, etc)
    - it is usable in `no_std` cfg
    - I guess `std` re-exports `core`
- macros
  - `assert`, `assert_eq`, and `assert_ne` assert
  - `cfg` evaluates cfg flags at compile time
  - `file`, `line`, and `column` expand to loc in the source file
  - `compile_error` generates a compile error
  - `concat` concatenates literals into `&'static str`
  - `dbg` prints and returns the value of an expression for quick debug
  - `debug_assert`, `debug_assert_eq`, and `debug_assert_ne` are asserts that
    are disabled in optimized builds by default
  - `env` and `option_env` return an envvar at compile time
  - `format` returns a `String`
  - `panic`, `todo`, `unimplemented`, `unreachable` all panic
  - `print` and `println` print to stdout
  - `eprint` and `eprintln` print to stderr
  - `write` and `writeln` print to a buffer
  - `stringify` returns `&'static str`
  - `thread_local` makes static decls thread-local
  - `vec` returns a `Vec`
- modules
  - `alloc` provides unsafe memory allocation
  - `any` is akin to C++ RTTI
  - `arch` provides arch-dependent cpu intrinsics
  - `array` provides utils to work with arrays
  - `ascii` provides utils for ascii (rather than utf8) strings
  - `backtrace` captures a stack backstrace
  - `borrow` provides traits such as `Borrow` or `ToOwned`
  - `boxed` provides `Box<T>`
  - `char`, `f32`, and `f64` should be considered internal
  - `cell` provides `Cell<T>` and variants
  - `clone` provides the `Clone` trait
  - `cmp` provides traits such as `Eq` or `Ord`, as well as utils
  - `collections` provides collection types
    - sequences: `Vec`, `VecDeque`, `LinkedList`
    - maps: `HashMap`, `BTreeMap`
    - sets: `HashSet`, `BTreeSet`
    - misc: `BinaryHeap`
  - `convert` provides traits such as `AsMut`, `AsRef`, `From`, `Into`, etc.
    - between `AsRef` and `From`
      - `AsRef` is essentially nop, e.g., `str` implements `AsRef<OsStr>`
      - `From` can have a cost, e.g., `String` implements `From<&str>`
  - `default` provides the `Default` trait
  - `env` works with process env, such as envvars, args, etc.
  - `error` provides the `Error` trait
  - `ffi` provides ffi bindings
    - c types: `c_void`, `c_char`, `c_int`, etc.
    - os strings: `OsStr`, `OsString`
      - rust strings, `str` and `String`, always store valid utf8 with
        explicit size
        - it does not rely on `\0` to terminate
        - `\0` is a valid utf8 char and can appear anywhere
        - there is no `\0` in the end
      - os strings, `OsStr` and `OsString`, always store valid wtf8 with
        explicit size
        - wtf8 is a superset of utf8, and as a result,
          - rust strings are os strings, and casting has no cost
          - os strings are not necessarily rust strings
            - casting requires no allocation but requires validation
        - this is done because the os can accept strings that are not valid utf8
        - on win, the os expects utf16 so it takes another conversion from os
          strings to to what win expects
    - c strings: `CStr`, `CString`
      - `CStr` and `CString` store any non-`\0` value and are `\0`-terminated
      - conversions between rust strings can fail in both directions
  - `fmt` provides string formatting traits and utils
    - `{}` formats with `Display` trait
    - `{:?}` formats with `Debug` trait
    - `{:x?}` formats with `Debug` trait, lower-case hex
    - `{:X?}` formats with `Debug` trait, upper-case hex
    - `{:o}` formats with `Octal` trait
    - `{:x}` formats with `LowerHex` trait
    - `{:X}` formats with `UpperHex` trait
    - `{:p}` formats with `Pointer` trait
    - `{:b}` formats with `Binary` trait
    - `{:e}` formats with `LowerExp` trait
    - `{:E}` formats with `UpperExp` trait
  - `fs` provides cross-os filesystem support
  - `future` is for `async` and `await`
  - `hash` provides traits such as `Hash`, `Hasher`, etc.
  - `hint` provides compiler hints
  - `io` provides cross-os io support
    - traits such as `Read`, `Write`, and `Seek`
      - also `BufRead` (read to an internal buffer first) and `IsTerminal`
    - funcs such as `stdin`, `stdout`, `stderr`
    - `Error` enum
  - `iter` provides traits and utils for iterations
    - traits such as `Iterator`, `DoubleEndedIterator`, and `ExactSizeIterator`
      - also `IntoIterator`, `FromIterator`
  - `marker` provides traits and types for basic props of types
    - traits such as `Copy`, `Send`, `Sized`, `Sync`,
    - `PhantomData`
  - `mem` provides utils about memory
    - `offset_of` macro
    - funcs such as `align_of`, `size_of`, `drop`, `swap`, `take`, `replace`
  - `net` provides cross-os tcp/udp support
  - `num` provides utils for numerical computation
  - `ops` provides operator overloading
    - traits such as `Add`, `Mul`, `Drop`, `Deref`, etc.
      - also `Fn`, `FnMut`, and `FnOnce` for callbacks
        - there are 3 kinds of func-like types
          - func items (that is, regular funcs)
          - closures
          - func pointers
        - they implement one or more of the traits
  - `option` provides `Option<T>`
  - `os` provides OS-specific utils
    - `linux` submodule, which is kinda empty
    - `unix` submodule, which extends `ffi`, `fs`, `io`, `net`, `process`,
      `thread`, etc.
    - `fd` submodule
      - types such as `RawFd`, `BorrowedFd`, `OwnedFd`
      - traits such as `AsFd`, `AsRawFd`, `FromRawFd`, `IntoRawFd`
    - `raw` submodule, which has been replaced by `core::ffi`
  - `panic` provides panic support
  - `path` provides cross-os path support
    - types such as `PathBuf` and `Path`
  - `pin` pins data to fixed addr
  - `prelude` uses a subset of std automatically
    - rust 2018
      - `std::borrow::ToOwned`
      - `std::boxed::Box`
      - `std::clone::Clone`
      - `std::cmp::{PartialEq, PartialOrd, Eq, Ord}`
      - `std::convert::{AsRef, AsMut, Into, From}`
      - `std::default::Default`
      - `std::iter::{Iterator, Extend, IntoIterator, DoubleEndedIterator, ExactSizeIterator}`
      - `std::marker::{Copy, Send, Sized, Sync, Unpin}`
      - `std::mem::drop`
      - `std::ops::{Drop, Fn, FnMut, FnOnce}`
      - `std::option::Option::{self, Some, None}`
      - `std::result::Result::{self, Ok, Err}`
      - `std::string::{String, ToString}`
      - `std::vec::Vec`
    - rust 2021
      - `std::convert::{TryFrom, TryInto}`
      - `std::iter::FromIterator`
    - other modules might have `prelude` submodules that can be used manually
      - `std::io::prelude`
      - `std::os::unix::prelude`
  - `primitive` re-exports primitive types which can be useful for
    macro-generated code
  - `process` provides cross-os process support
    - structs such as `Command` and `Child`
    - funcs such as `abort` and `exit`
  - `ptr` provides mostly unsafe utils for raw pointers
  - `rc` provides `Rc<T>` and `Weak<T>`
  - `result` provides `Result<T>`
  - `slice` provides utils to slice
  - `str` provides utils to `str`
  - `string` provides `String`
  - `sync` provides structs such as `Arc`, `Mutex`, etc.
  - `task` provides structs and traits for async tasks
  - `thread` provides cross-os native thread support
    - types such as `Builder`
    - funcs such as `current`, `park`, `sleep`, `spawn`, `yield_now`
  - `time` provides structs such as `Duration`, `Instant` (`CLOCK_MONOTONIC`),
    `SystemTime` (`CLOCK_REALTIME`)
  - `vec` provides `Vec<T>`
