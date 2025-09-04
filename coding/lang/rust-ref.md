The Rust Reference
==================

## Chapter 2. Lexical structure

- 2.1. Input format
- 2.2. Keywords
- 2.3. Identifiers
- 2.4. Comments
- 2.5. Whitespace
- 2.6. Tokens

## Chapter 3. Macros

- 3.1. Macros By Example
- 3.2. Procedural Macros

## Chapter 4. Crates and source files

- each compilation processes a single crate in the source form and produces
  the crate in the binary form (executable or library)
- a crate is a unit of compilation and linking
  - it contains a tree of modules
  - the root module is anonymous
- the compiler is always invoked with a single source file
  - the source file is the root module of a crate
  - it can reference external source files
  - each file is a module
    - but not every module has its own file
- each source file contains zero or more items
  - it may optionally begin with attrs
- a crate that contains a `main` function can be compiled to an executable
  - the root module must define `main`, or have `use foo::bar as main`
  - the main function must
    - have no args
    - have no trait/lifetime bounds
    - have no where clauses
    - have return type that implements `std::process::Termination`
      - such as `()`, `!`, `Infallible`, `ExitCode`
      - also `Result<T, E> where T: Termination, E: Debug`

## Chapter 5. Conditional compilation

- source code can be conditionally compiled using
  - `cfg` attr
  - `cfg_attr` attr
  - `cfg` macro
  - each form takes a predicate that evaludates to true or false
  - a predicate is one of
    - a config option
      - a name or a key-val pair
    - `all(opt1,...)`
    - `any(opt1,...)`
    - `not(opt)`
- config options
  - arbitrary options
    - `rustc --cfg foo` sets `foo`
    - cargo specifies `--cfg feature` for example
  - target options
    - `target_arch`, with values such as
      - `x86`, `x86_64` `arm`, `aarch64`, etc.
    - `target_feature`, with values such as
      - `avx`, `avx2`, `rdrand`, etc.
    - `target_os`, with values such as
      - `windows`, `macos`, `ios`, `linux`, `android`, `freebsd`, etc.
      - `none` for embedded targets
    - `target_family`, with values such as
      - `unix`, `windows`, `wasm`, etc.
    - `target_env`, with values such as
      - usually empty, but can be `gnu`, `msvc`, `musl`, etc.
    - `target_abi`, with values such as
      - usually empty, but can be `llvm`, etc.
    - `target_endian`, with values `little` or `big`
    - `target_pointer_width`, with values `16`, `32`, or `64`
    - `target_vendor`, with values such as
      - `apple`, `pc`, `unknown`, etc.
    - `target_has_atomic`, with values such as
      - `8`, `16`, `34`, `64`, `128`, `ptr`, etc.
  - when `target_family` is `unix` or `windows`, shorthands are set as well
    - `unix` if unix
    - `windows` if windows
  - other options
    - `test`, if rustc is invoked with `--test`
    - `debug_assertions`, if compiling without optimization
    - `proc_macro`, if the crate type is `proc_macro`
    - `panic`, with values such as
      - `abort`, `unwind`, etc.

## Chapter 6. Items

- 6.1. Modules
- 6.2. Extern crates
- 6.3. Use declarations
- 6.4. Functions
- 6.5. Type aliases
- 6.6. Structs
- 6.7. Enumerations
- 6.8. Unions
- 6.9. Constant items
- 6.10. Static items
- 6.11. Traits
- 6.12. Implementations
- 6.13. External blocks
- 6.14. Generic parameters
- 6.15. Associated Items

## Chapter 7. Attributes

- syntax
  - `#![attr]` applies to the item that the attr declared within
    - when declared inside a module, it applies to the module
  - `#[attr]` applies to the item that follows
- built-in attributes
  - Conditional compilation
    - `cfg` Controls conditional compilation.
    - `cfg_attr` Conditionally includes attributes.
  - Testing
    - `test` Marks a function as a test.
    - `ignore` Disables a test function.
    - `should_panic` Indicates a test should generate a panic.
  - Derive
    - `derive` Automatic trait implementations.
    - `automatically_derived` Marker for implementations created by derive.
  - Macros
    - `macro_export` Exports a macro_rules macro for cross-crate usage.
    - `macro_use` Expands macro visibility, or imports macros from other
      crates.
    - `proc_macro` Defines a function-like macro.
    - `proc_macro_derive` Defines a derive macro.
    - `proc_macro_attribute` Defines an attribute macro.
  - Diagnostics
    - `allow`, `warn`, `deny`, `forbid` Alters the default lint level.
    - `deprecated` Generates deprecation notices.
    - `must_use` Generates a lint for unused values.
    - `diagnostic::on_unimplemented` Hints the compiler to emit a certain
      error message if a trait is not implemented.
  - ABI, linking, symbols, and FFI
    - `link` Specifies a native library to link with an extern block.
    - `link_name` Specifies the name of the symbol for functions or statics in
      an extern block.
    - `link_ordinal` Specifies the ordinal of the symbol for functions or
      statics in an extern block.
    - `no_link` Prevents linking an extern crate.
    - `repr` Controls type layout.
    - `crate_type` Specifies the type of crate (library, executable, etc.).
    - `no_main` Disables emitting the main symbol.
    - `export_name` Specifies the exported symbol name for a function or
      static.
    - `link_section` Specifies the section of an object file to use for a
      function or static.
    - `no_mangle` Disables symbol name encoding.
    - `used` Forces the compiler to keep a static item in the output object
      file.
    - `crate_name` Specifies the crate name.
  - Code generation
    - `inline` Hint to inline code.
    - `cold` Hint that a function is unlikely to be called.
    - `no_builtins` Disables use of certain built-in functions.
    - `target_feature` Configure platform-specific code generation.
    - `track_caller` Pass the parent call location to
      `std::panic::Location::caller()`.
    - `instruction_set` Specify the instruction set used to generate a
      functions code
  - Documentation
    - `doc` Specifies documentation. See The Rustdoc Book for more
      information. Doc comments are transformed into `doc` attributes.
  - Preludes
    - `no_std` Removes `std` from the prelude.
    - `no_implicit_prelude` Disables prelude lookups within a module.
  - Modules
    - `path` Specifies the filename for a module.
  - Limits
    - `recursion_limit` Sets the maximum recursion limit for certain
      compile-time operations.
    - `type_length_limit` Sets the maximum size of a polymorphic type.
  - Runtime
    - `panic_handler` Sets the function to handle panics.
    - `global_allocator` Sets the global memory allocator.
    - `windows_subsystem` Specifies the windows subsystem to link with.
  - Features
    - `feature` Used to enable unstable or experimental compiler features. See
      The Unstable Book for features implemented in rustc.
  - Type System
    - `non_exhaustive` Indicate that a type will have more fields/variants
      added in future.
  - Debugger
    - `debugger_visualizer` Embeds a file that specifies debugger output for a
      type.
    - `collapse_debuginfo` Controls how macro invocations are encoded in
      debuginfo.
- 7.1. Testing
- 7.2. Derive
- 7.3. Diagnostics
- 7.4. Code generation
- 7.5. Limits
- 7.6. Type System
- 7.7. Debugger

## Chapter 8. Statements and expressions

- 8.1. Statements
  - decl stmts
    - it introduces one or more names to the enclosing block
    - item decls
      - same as item decls at module level
      - can be used to declare new uses, structs, fns, etc.
    - let stmts: `let pat;` or `let pat = expr;`
      - in full form, `let pat: type = expr else block;`
  - expr stmts: `expr;` or `block;`
    - it evaluates an expr and ignores the result
      - it wants the side-effect of the evalution
    - `block;` is a stmt
      - `block` is an expr and must evaluate to `()`
      - in the context where a stmt is allowed, the semicolon can be omitted
        - it becomes `block` but is a stmt rather than an expr
      - for comparison, `let v = block;` does not permit stmt
        - `block` is an expr and can evaluate to any value
- 8.2. Expressions
  - precedence
    - paths
    - method calls
    - fields
    - function calls, array indexing
    - `?`
    - unary ops: `-`, `*`, `!`, `&`, and `&mut`
    - `as`
    - arith ops
      - `*`, `/`, `%`
      - `+` and `-`
    - logical ops
      - `<<` and `>>`
      - `&`
      - `^` 
      - `|` 
    - cmp ops: `==`, `!=`, `<`, `>`, `<=`, and `>=`
    - lazy boolean ops
      - `&&`
      - `||`
    - range ops: `..` and `..=`
    - assignments: `=`, `+=`, `-=`, `*=`, `/=`, `%=`, `&=`, `|=`, `^=`, `<<=`, and `>>=`
    - `return`, `break`, closures
  - exprs are divided into 2 major and 1 minor categories
    - place exprs: `var`, `*var`, `var[3]`, `var.field`, etc.
      - they represent memory locations
      - aka lvalues
    - value exprs: the rest
      - aka rvalues
    - assignee exprs: lhs of an assignment expr; the allowed exprs are
      - place exprs
      - tuples, slices, tuple structs, or structs of place exprs
      - `_` and `()`
  - exprs are evaluated in 2 contexts
    - place context: `&var`, `*var`, `var.field`, `var[3]`, rhs of `let`,
      etc.
    - value context: the rest
  - when a place expr is evaluated in the value context, it denotes the value
    held in that memory location
    - the value is copied if its type implements `Copy`
    - the value is moved if the place expr is
      - a variable that is not currently borrowed
      - temporary variable
      - more
    - otherwise, it is an error
  - Literal expressions
    - `true`, `false`, char/byte/string literals, numeric literals, etc.
    - `b'x'` is `u8`
    - `b"x"` is `[u8; N]`
    - `c"x"` is `CStr`
    - `123u64` is `u64`
  - Path expressions
  - Block expressions
    - `{ stmts; optinal expr }` is a block
    - `const { ... }` evaluates at compile-time
    - `unsafe { ... }` is an unsafe block
  - Operator expressions
    - unary
      - borrow ops: `&expr`, `&mut expr`
      - raw addr-of ops: `ptr::addr_of!(expr)`
      - deref op: `*expr`
        - if `x` is a reference, `*x` becomes `*std::ops::Deref::deref(&x)`
        - if `x` is a raw pointer, `*x` is unsafe
      - question mark op: `expr?`
        - if `x` is a `Result<T, E>`, it either unwraps or return
          `Err(From::from(e))`
        - if `x` is a `Option<T>`, it either unwraps or return `None`
      - negation ops: `-expr`, `!expr`
    - binary
      - arith ops
        - `expr1 + expr2`
        - `expr1 - expr2`
        - `expr1 * expr2`
        - `expr1 / expr2`
        - `expr1 % expr2`
        - `expr1 & expr2`
        - `expr1 | expr2`
        - `expr1 ^ expr2`
        - `expr1 << expr2`
        - `expr1 >> expr2`
      - comparison ops
        - `expr1 == expr2`
        - `expr1 != expr2`
        - `expr1 > expr2`
        - `expr1 < expr2`
        - `expr1 >= expr2`
        - `expr1 <= expr2`
      - lazy bool ops
        - `expr1 && expr2`
        - `expr1 || expr2`
      - Type cast expressions
        - `as` supports all allowed coercions, plus more
        - numeric to numeric
          - e.g., `i32` to `f32`, `i64` to `i32`, etc.
        - enum to integer (only when the enum is plain)
        - `bool` or `char` to integer
        - `u8` to `char`
        - `*T` to `*U` (reinterpret cast)
        - `*T` to integer (can truncate)
        - integer to `*T`
        - `&T` to `*T` (but no const cast)
        - `&[T; n]` to `*T` (array to pointer)
        - function item to `*T` (because the type does not matter) or integer
        - function pointer to `*T` (because the type does not matter) or integer
        - non-capturing closures to function pointers
      - assignment expressions
        - `expr1 = expr2`
      - compound assignment expressions
        - `expr1 += expr2`
        - `expr1 -= expr2`
        - `expr1 *= expr2`
        - `expr1 /= expr2`
        - `expr1 %= expr2`
        - `expr1 &= expr2`
        - `expr1 |= expr2`
        - `expr1 ^= expr2`
        - `expr1 <<= expr2`
        - `expr1 >>= expr2`
  - Grouped expressions
    - `(expr)`
  - Array and index expressions
    - array exprs: `[expr1, ...]` or `[expr; N]`
    - array/slice indexing exprs: `expr1[expr2]`
  - Tuple and index expressions
    - tuple exprs: `(expr1, ...)`
    - tuple indexing exprs: `expr1.index`
  - Struct expressions
    - field struct expr: `Name{fields}`
    - tuple struct expr: `Name(types)`
    - unit struct expr: `Name`
  - Call expressions
    - `expr(params)`
  - Method call expressions
    - `expr.method(params)`
  - Field access expressions
    - `expr.field`
  - Closure expressions
    - `|params| expr`
    - `|params| -> type { expr }`
    - prefixing `move` to capture by moves rather than borrows
  - Loop expressions
    - infinite loops: `loop block`
    - predicate loops: `while expr block`
    - predicate pattern loops: `while let pattern = expr block`
      - same as `loop { match expr { pattern => block, _ => break, } }`
    - iterator loops: `for pattern in expr block`
      - roughly
        - `let mut iter = std::iter::IntoIterator::into_iter(expr);`
        - `while let Some(val) = std::iter::Iterator::next(&mut iter) {`
        - `  let pattern = val;`
        - `  block;`
        - `}`
      - note that `IntoIterator::into_iter` takes ownership
    - loop labels: `'label:`
    - break exprs: `break 'label expr`
    - continue exprs: `continue 'label`
  - Range expressions
    - `start..end` has type `std::ops::Range`
    - `start..` has type `std::ops::RangeFrom`
    - `..end` has type `std::ops::RangeTo`
    - `..` has type `std::ops::RangeFull`
    - `start..=end` has type `std::ops::RangeInclusive`
    - `..=end` has type `std::ops::RangeToInclusive`
  - If and if let expressions
    - `if expr block else block`
    - `if let pattern = expr block else block`
  - Match expressions
    - `match expr { arm1 => expr1, arm2 => block2 }`
  - Return expressions
    - `return expr`
  - Await expressions
    - `expr.await`
  - Underscore expressions
    - `_`

## Chapter 9. Patterns

- patterns are used to match values inside structures and, optionally,
  bind variables to those values
- patterns are used in
  - `let`
  - func and closure params
  - `match`
  - `if let`
  - `while let`
  - `for`
- patterns can destructure structs, enums, and tuples
  - `_` is a placehold and matches a single field
  - `..` is a wildcard and matches remaining fields
- patterns can be refutable (can fail) or irrefutable (never fails)
  - `Some(x)` in `if let Some(x)...` is refutable while `(x, y)` in
    `let (x, y)...` is irrefutable
- literal patterns
  - `true`, `false`, char/byte/string literals, numeric literals, etc.
- identifier patterns
  - identifier, optionally prefixed by `ref` and `mut`
    - `mut x` in `let mut x = 10;`
    - `x` in `fn foo(x: i32)`
- binding modes
  - in `let x = String::from("test");`, the identifier `x` binds to the string
    - the binding is a copy, if the val impls `Copy`
    - otherwise, the binding is a move
      - this is the case with the string here
    - we can also force the binding to be a reference by prefixing the
      identifier with `ref`
  - when the compiler matches a pattern, it starts from outside the pattern
    and works inward
    - the default binding mode is copy/move at the beginning and may change in
      the process
    - e.g., when it matches `Some(x)` against `Some(3)`,
      - `Some` matches `Some` and the default binding mode is not changed
      - `x` matches 3 and binds by copy
    - but when it matches `Some(x)` against `&Some(3)`,
      - `&Some` is automatically dereferenced to `Some`, and the default
        binding mode is changed to reference
      - `Some` matches `Some`
      - `x` matches 3 and binds by reference
- wildcard pattern
  - `_` matches any value
  - when used inside another pattern, it matches a single data field while
    `..` matches the remaining fields
- range patterns
  - `..=10` matches values in `(.., 10]`
  - `5..=10` matches values in `[5, 10]`
  - `10..` matches values in `[10, ..)`
  - the type of the pattern is determined by its bounds
- reference patterns
  - `&pattern` matches a reference and dereferences it
  - e.g.,
    - `let a = 3; assert_eq!(a, 3);`
    - `let b = &3; assert_eq!(*b, 3);`
    - `let ref c = 3; assert_eq!(*c, 3);`
    - `let ref d = &3; assert_eq!(**d, 3);`
    - `let &e = &3; assert_eq!(e, 3);`
      - this works because `3` impls `Copy`
    - `let &f = &Box::new(3);` does not compile
      - `let x = Box::new(3); let y = &x; let f = *y;`
- struct patterns
  - `Type{...}` matches struct, enum, and union values
- tuple struct patterns
  - `Type(...)` matches tuple struct and enum values
- tuple patterns
  - `(...)` matches tuple values
- grouped patterns
  - `(pattern)` to explicitly control precedence of compound patterns
- slice patterns
  - `[...]` matches arrays and slices
- path patterns
- constant patterns
- or-patterns
  - `pat1 | pat2` matches either patterns

## Chapter 10. Type system

- 10.1. Types
  - built-in types
    - primitive types
      - boolean: `bool`
      - numeric: `{u,i}{8,16,32,64,128}` and `f{32,64}`
      - textual: `char` and `str`
        - `char` is `u32`, with valid UTF-32 values excluding surrogates
        - `str` is `[u8]`, with valid UTF-8 values
      - never: `!`
        - can only be used as the return type (of a noreturn func)
    - sequence types
      - tuple: `(T...)`
        - `()` is allowed and is a zero-sized type (ZST)
          - it is called unit
      - array: `[T; N]`
      - slice: `[T]`
        - dynamically-sized type (DST)
          - not `Sized` and cannot be instantiated
          - this is because the slice len is unknown
        - instantiated as pointer types such as `&[T]`
          - pointer types to DST are `Sized`, but is twice the size of pointer
            types to `Sized`
          - also known as fat pointers
          - the first pointer points to the underlying data
          - the second pointer points to the end of the data
        - iow,
          - these types are different and are `Sized`
            - `[T; 5]`
            - `[T; 10]`
            - `Vec<T>`
          - they have something in common, and that something is `[T]` which
            is not `Sized`
          - we can instead get `&[T]` from them and operate on `&[T]`
        - ways to get a slice from a vec
          - `v.as_slice()`
          - `&v[..]`
          - because vec implements `Deref<Target = [T]>`
            - `&*v`
            - `&Vec<T>` also implicitly coerces to `&[T]`
          - because vec implements `AsRef<[T]>`,
            - `v.as_ref()`
            - this is rarely used
              - when a function takes `impl AsRef<[T]>` instead of `&[T]`, it
                can also support `str`, `String`, etc.
        - `str` is `[u8]` whose data are guaranteed to be valid utf8
    - user-defined types
      - struct: `struct Name ...`
        - `struct Name;` or `struct Name{};` are allowed and are ZST
        - `struct Name(T...)` is allowed and is called a tuple struct
      - enum: `enum Name ...`
        - each enum item is like a struct, including empty struct
      - union: `union Name ...`
        - always unsafe to read
    - function types
      - function items (regular functions): `fn name ...`
        - always ZST
        - always implements
          - `Fn` (implies `FnMut` and `FnOnce`)
          - `Copy` (implies `Clone`)
          - `Send` and `Sync`
      - closures: `|...| ...`
        - a closure is approximately a struct
          - the captured variables become struct fields
          - it implements at least `FnOnce`, where the implementation is the
            closure body
          - it may implement `FnMut` or `Fn`, depending on the closure body
    - pointer types
      - references: `&T` and `&mut T`
      - raw pointers: `*const T` and `*mut T`
        - always unsafe to dereference
      - function pointers: `fn ...`
        - say we have two functions, `fn test1(){}` and `fn test2(){}`
        - the two functions are different types
          - `let mut a = test1; a = test2;` fails
          - `let mut a = &test1; a = &test2;` fails
        - the two functions can be coerced to the same function pointer type
          - `let mut a: fn(); a = test1; a = test2; a = ||{}` works
            - because functions and non-capturing closures can be coerced to
              function pointers
          - `let mut a: fn(); a = &test1;` fails
            - because a ref to a function is not the same nor cocerced to a
              function pointer
        - always implements `Fn` (implies `FnMut` and `FnOnce`)
    - trait types
      - trait objects: `dyn Trait...`
        - dynamically-sized type (DST)
          - not `Sized` and cannot be instantiated
          - this is because the concrete type is unknown
        - instantiated as pointer types such as `&dyn Trait`
          - pointer types to DST are `Sized`, but is twice the size of pointer
            types to `Sized`
          - also known as fat pointers
          - the first pointer points to an instance of the concrete type
          - the second pointer points to a vtable
        - note that a trait itself, `trait Trait ...`, is not a type
      - impl trait: `impl Trait`
        - `fn foo(arg: impl Trait)` is approximately
          `fn foo<T: Trait>(arg: T)`, but different
        - can only be used as the param or return type of a func
- 10.2. Dynamically Sized Types
- 10.3. Type layout
- 10.4. Interior mutability
- 10.5. Subtyping and Variance
- 10.6. Trait and lifetime bounds
  - trait and lifetime bounds are provided in a `where` clause
  - there are syntax sugars
    - `fn foo<T: Copy>` is `fn foo<T> where T: Copy`
      - it means `T` must implement `Copy`
    - `trait Foo: Copy` is `trait Foo where Self: Copy`
      - it means, to implement `Foo`, the type must implement `Copy` too
    - `trait Foo { type Bar: Copy; ... }` is
      `trait Foo where Self::Bar: Copy { type Bar; ... }`
      - it means `Bar` must implement `Copy`
  - `Sized` trait
    - `Sized` trait bound is implicit for generics and associated types
    - add `?Sized` as a trait bound to remove it
    - add `!Sized` to negate it
  - lifetime bounds
    - `fn foo<'a, 'b> where 'a: 'b` means `'a` outlives `'b`
    - `fn foo<'a, T> where T: 'a` means all lifetime parameters of `T` outlive
      `'a`
      - `T: 'static` means `T` is not borrowed
- 10.7. Type coercions
  - implicit type conversions
  - allowed type coercions
    - `T` to `U`, if `T` is a subtype of `U`
      - this usually means `T` and `U` are the same type with different
        lifetimes
      - e.g., `&'static str` is a subtype of `&str`
    - `T1` to `T3`, if `T1` coerces to `T2` and `T2` coerces to `T3`
    - `&mut T` to `&T`
    - `*mut T` to `*const T`
    - `&T` to `*const T`
    - `&mut T` to `*mut T`
    - (`&T` or `&mut T`) to `&U`, if `impl Deref for T { type Target = U; }`
      - this usually means `T` wraps `U` and can be used as `U`
      - e.g., `Box<T>` derefs to `&T`, `String` derefs to `str`
    - `&mut T` to `&mut U`, if `impl Deref for T { type Target = U; }`
    - `TyCtor(T)` to `TyCtor(U)`, if `T` unsized-coerces to `U`
      - `TyCtor(T)` is one of
        - `&T`, `&mut T`
        - `*const T`, `*mut T`
        - `Box<T>`
      - unsized coercions convert sized types to unsized types, such as
        - `[T; n]` to `[T]`
        - `T` to `dyn U`, if `T` implements `U + Sized` and `U` is
          object safe (i.e., can be represented by a vtable)
    - function item types (regular functions) to `fn` pointers
    - non-capturing closures to `fn` pointers
    - `!` to `T`
  - there is also the type cast operator, `as`
- 10.8. Destructors
  - temporary lifetime extension
    - a temp is usually dropped at the end of the stmt, `foo(&vec!(1));`
    - but when a temp is referenced, its lifetime is extended
      - `let a = &vec!(1);` is the same as
        `let _tmp = vec!(1); let a = &_tmp;`

- 10.9. Lifetime elision

## Chapter 11. Special types and traits

## Chapter 12. Names

- 12.1. Namespaces
- 12.2. Scopes
- 12.3. Preludes
- 12.4. Paths
- 12.5. Name resolution
- 12.6. Visibility and privacy

## Chapter 13. Memory model

- 13.1. Memory allocation and lifetime
  - an item of a program is a value calculated at compile-time, stored in the
    binary, and mapped in the process
    - it is neither dynamically allocated nor freed
    - e.g., modules, functions, static/const variables, etc.
  - the heap is a region for box allocations
    - a box value allocates from the heap
      - e.g., `let a = Box::new(3)`
      - the box value is an addr stored in the stack
      - the allocation is an `i32` in the heap
    - the box value and its heap allocation have the same lifetime
- 13.2. Variables
  - a variable is a component of a stack frame
    - a named function parameter
    - a named local variable
    - an anonymous temporary
  - on frame entry, an allocation is made from the stack
    - local variables are stored in that allocation, and are uninitialized

## Chapter 14. Linkage

- each compilation processes a crate in the source form and produces the
  crate in binary form(s)
- `--crate-type=<type>` or `#![crate_type = "<type>"]` specifies the binary
  form(s)
  - `bin` produces an executable
    - there must be a `main` function
  - `lib` produces a library
    - it aliases one of other types below, determined by the compiler
  - `dylib`/`rlib` produces a dynamic/static rust library
    - it contains metadata and can be used as a dep by other crates
  - `cdylib`/`staticlib` produces a dynamic/static system library
    - it is for ffi
  - `proc-macro`
- dependencies
  - for `staticlib`, deps must be `rlib`
  - for `rlib`, deps can be `rlib` or `dylib`
    - `rlib` references its deps and does not own copies of its deps
  - for `bin`, `rlib` is attempted first before `dylib`
  - for `dylib`, deps can be `rlib` or `dylib`
    - `rlib` owns copies of its deps
- statically-linked executables
  - both static linking and dynamic linking of C runtime may be supported
    - it depends on the targets
    - `x86_64-unknown-linux-gnu` defaults to dynamic linking
    - `x86_64-unknown-linux-musl` defaults to static linking
  - `-C target-feature=[+-]crt-static` to enables/disables static linking

## Chapter 15. Inline assembly

## Chapter 16. Unsafety

- 16.1. The unsafe keyword
- 16.2. Behavior considered undefined
- 16.3. Behavior not considered unsafe

## Chapter 17. Constant Evaluation

## Chapter 18. Application Binary Interface

## Chapter 19. The Rust runtime

## Chapter 20. Appendices

- 20.1. Macro Follow-Set Ambiguity Formal Specification
- 20.2. Influences
- 20.3. Glossary
