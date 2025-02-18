LLVM
====

## Build LLVM

- build just llvm
  - `git clone https://github.com/llvm/llvm-project.git`
  - `cmake -Sllvm -Bout -GNinja -DCMAKE_BUILD_TYPE=Debug`
  - this builds just LLVM
    - `bin`: `llvm-*`, `llc`, `lli`, `opt`
    - `include`: `llvm`, `llvm-c`
    - `lib`
      - `cmake`
      - `libLLVM*.a`
      - `libLTO.so`, for use by linkers as a plugin
      - `libRemarks.so`, for parsing diagnostics emitted by llvm in object
        files
    - `share`: `opt-viewer`
- build stage1 compiler
  - `-DCMAKE_BUILD_TYPE=Release`
  - `-DLLVM_TARGETS_TO_BUILD="X86;ARM;AArch64"`
  - `-DLLVM_ENABLE_PROJECTS="clang;lld"`
  - `-DLLVM_ENABLE_RUNTIMES="compiler-rt;libcxx;libcxxabi"`
    - runtimes are just projects that can be listed in `LLVM_ENABLE_PROJECTS`
    - the difference is runtimes are built with the newly built compiler while
      projects are built with the host compiler
  - this builds llvm plus
    - `bin`
      - `clang*`, `scan-*`, etc from `clang`
      - `lld` from `lld`
    - `include`
      - `c++` from `libcxx`
      - `clang` and `clang-c` from `clang`
      - `lld` from `lld`
      - `x86_64-unknown-linux-gnu/cxx` from `libcxx`
    - `lib`
      - `libclang*.a` from `clang`
      - `libclang.so` and `libclang-cpp.so` from `clang`
      - `liblld*.a` from `lld`
      - `clang/` from `clang` and `compiler-rt`
      - `libear/` from `clang`
      - `libscanbuild/` from `clang`
      - `x86_64-unknown-linux-gnu/`
        - `libc++.a` and `libc++.so*` from `libcxx`
        - `libc++abi.a` and `libc++abi.so*` from `libcxxabi`
    - `libexec`
        - `{analyze,intercept}-{cc,c++}` from `clang`
        - `{ccc,c++}-analyzer` from `clang`
  - the compiler supports
    - `x86_64-unknown-linux-gnu`
    - `i686-unknown-linux-gnu`
    - `aarch64-unknown-linux-gnu`
- build stage2 compiler
  - to build the same stuff using the stage1 compiler

## Build for Mesa CLC

- build
  - `git clone https://github.com/llvm/llvm-project.git`
  - `cd llvm-project`
  - `git -C llvm/projects clone https://github.com/KhronosGroup/SPIRV-LLVM-Translator.git`
  - `git -C llvm/projects clone https://github.com/KhronosGroup/SPIRV-Headers.git`
  - `cmake -Sllvm -Bout -GNinja \
         -DCMAKE_BUILD_TYPE=Debug \
         -DCMAKE_INSTALL_PREFIX="$PWD/install" \
         -DCMAKE_C_COMPILER_LAUNCHER=ccache \
         -DCMAKE_CXX_COMPILER_LAUNCHER=ccache \
         -DLLVM_BUILD_LLVM_DYLIB=ON \
         -DLLVM_ENABLE_PROJECTS="clang;libclc" \
         -DLLVM_TARGETS_TO_BUILD= \
         -DLLVM_ENABLE_RTTI=ON \
         -DLLVM_LINK_LLVM_DYLIB=ON \
         -DLIBCLC_TARGETS_TO_BUILD="spirv-mesa3d-;spirv64-mesa3d-" \
         -DLLVM_SPIRV="$PWD/out/bin/llvm-spirv"`
  - `ninja -C out install`
- mesa
  - `LLVM_CONFIG=<install>/bin/llvm-config`
  - `PKG_CONFIG_PATH=<install>/lib/pkgconfig`
  - `LD_LIBRARY_PATH=<install>/lib`

## Debian Packaging

- `libllvm$ver` for `libLLVM-$ver.so.1`
- `llvm-$ver-dev` for c++ headers, c headers, cmake rules, static libraries,
  shared library links
- `llvm-$ver` for tools such as `llc-$ver`, `llvm-config-$ver`, etc.
- `llvm-$ver-linker-tools` for linker plugins
- `llvm-$ver-runtime` for tools such as `lli-$ver`
- `llvm-$ver-tools` for tools such as `FileCheck-$ver`

## Compiler Steps

- source to AST
  - lex and parse
  - LR parser
  - LALR parser
  - LL parser
  - recursive descent parser
    - usually hand-written
  - operator precedence parser
    - make adding operators dynamically easy
- AST to IR
- IR to machine code
  - instruction selection
  - instruction scheduling
  - register allocation

## IR syntax

- `@` global; `%` local
- global variables are always pointers
  - `@G = weak global i32 0` has type `i32*`
  - just like the type of `%g = alloca i32`
- thank to the prefix, no reserve words (like instrutions or types) will
  conflict with identifiers
- `;` starts a comment
- `Module` is a translation unit (like a C file).  It consists of functions,
   global variables, and symbol table entries.
  - two modules can be merged together by the linker
  - basically, global variables and functions are global values
- Global values
  - linkage types
  - calling conventions
  - visibility (default/hidden/protected)
- named type
  - `%mytype = type { %mytype*, i32 }`
- function
  - there are basic blocks
  - the first one is special in that no block can branch to it
  - blocks must be terminated
- alias
  - `@name2 = alias type 1 @name1`
- function attributes
  - inline, noreturn, readnone
- inline assembly
  - `module asm "blah"`


## LLVM Tutorial

- <http://www.llvm.org/docs/tutorial/>
- Lexer
  - convert the source file into tokens and metadata (like numeric values)
- AST classes
  - Abstract Syntax Tree
  - AST is the output of the parser
  - there are expressions
    - numeric Expr, variable Expr, binary Expr, call Expr
  - there are also Func and Proto
    - Func is Proto plus Expr
- Parser
  - a recursive function `ParseExpression`.
    - Recursive Descent Parsing
  - how to parse `( expr )`?
    - eat `(`
    - call `ParseExpression`
    - eat `)` and bail if no `)`
  - how to parse binary expression?
    - Operator-Precedence Parsing
    - Consider `A+B*C-D`, we want to parse it to `(-, LHS, RHS)`, where RHS is `D`
      and LHS is `(+, A, B*C)`.
    - We can have `ParseBinOpRHS(precedence, LHS)`
      - it is called with precedence 0 (lower than _any_ op), and LHS `A`
      - it needs to parse RHS and merge both side with current op
      - and loop
      - precedence controls how RHS is parsed
        - e.g., for LHS `A`, the RHS is `B*C`, not `B*C-D`
- codegen
  - define a virtual codegen method in each AST Expr to generate IRs
  - the return value of codegen method is `llvm::Value *`
    - it represents a SSA register (aka SSA value).
    - the most distinct aspect of an SSA value is that its real value is
      computed as the related instruction executes; it does not get a new
      real value until the instruction re-executes (like in a loop)
  - to generate numeric float
    - `ConstantFP::get(getGlobalContext(), APFloat(val))`
    - FP stands for floating point; AP stands for arbitrary precision
  - expressions are straightforward
    - operators map to instructions directly
  - Func AST
    - A function returns a Value, derived from its arguments
    - It is a prototype with `BasicBlock`.
- Optimization
  - constants are folder when IRs are built through IR builder
  - sophiscated optimizations are done in the form of passes
  - For per-function opt, one can create a `FunctionPassManager` and run every
    function through it.  The functions will be optimized in place.
- JIT
  - One needs to create an `ExecutionEngine`, which can be a JIT compiler or
    the LLVM interpreter.  The latter is a fallback.
  - Unresolved symbols will be `dlsym()`ed.  Or looked up through user-defined
    method.
- Control Flow
  - An if/then/else expression
    - terminate current block with conditional branch
    - create and terminate `then` block
    - create and terminate `else` block
    - create `ifcont` block but does not terminate it
    - return Phi node as the value computed by the expression
  - Phi function
    - Every variable is versioned
    - an if/then/else changes X depending on the condition, which gives two
      versions of X
    - if X is used after if/then/else, which version of X to use?
    - Use Phi!
    - Both `then` and `else` are `BasicBlock`s.  Phi basically says, if from
      block A, use this version; if from block B, use that version.
  - Every block needs to be terminated (with `ret`, `br`, and etc.)
  - Nested if/then/else
    - A nested if/then/else terminates current block and creates a new
      unterminated blcok
    - As parent if/then/else, the only thing to do is to `GetIntertBlock`
      to get current block after the child if/then/else.
  - A loop expression
    - terminate current block with a branch
    - create and terminate `loop` block
    - create `afterloop` block but does not terminate it
    - return constant 0.0.
- Mutable Variable
  - LLVM IR is by nature SSA
  - it makes `int x = 3; x = x + 1;` non-trivial.
    - because there are two versions of `x`
  - for imperative language, the front-end needs to do SSA construction
    - but it is not recommended
  - a simple solution is to use `alloca/load/store` to manipulate `x`
  - but it is slow
  - run `mem2reg` pass to optimize it!

## IR Overview

- Type classifications
  - integer
  - floating point
  - first class: integer, floating point, pointer, vector, structure, array, label, metadata
  - primitive: label, void, floating point, metadata
  - derived: integer, array, function, pointer, structure, packed structure, vector, opaque
- Instruction classifications
  - terminator: ret, br, switch (jump table), ...
  - binary: add, fadd, rem (remainder), ...
  - bitwise: shl, lshr, ashr, or, and, xor
  - vector: extractelement, insertelement, shufflevector
  - aggregate (struct or array): extractvalue, insertvalue
  - memory: alloca, load, store, getelementptr
  - conversion: trunc, zext, fptrunc, fptoui, uitofp, ptrtoint, bitcast, ...
  - other: icmp, fcmp, phi, select (?:), call, `va_arg`
- Intrinsics
  - comparing to intructions, intrinsics can be added without changing bitcode
    reader, parser, and etc.
  - intrinsics are external functions.  they can only be `call`ed or `invoked`.
  - new extensions are implemented as intrinsics before turning into
    instructions.
- Intrinsics Classifications
  - variable arguments: `va_start`, `va_end`, `va_copy`
  - garbage collection: gcroot, gcread, gcwrite
  - code generator: returnaddress, ...
  - libc: memcpy, sin, cos, pow, ...
  - bit manipulation: bswap, ctpop (count population), ctlz, cttz
  - arithmetic with overflow:
  - debugger:
  - exception:
  - trampoline:
  - atomic:
  - memory use marker:
  - general:

## Passes

- Classifications
  - Analysis
  - Transform
  - Utility

## Libraries

- `VMCore` provides instructions and the type system
- `Archive`, `AsmParser`, and `Bitcode` reads IR from files
- `Support` helps build IR, command-line parsing, and etc.
- `System` provides OS-related support
  - <http://www.llvm.org/docs/SystemLibrary.html>
- `Analysis` and `Transforms` provides passes
- `CodeGen` is target-independent code generator
  - <http://llvm.org/docs/CodeGenerator.html>
- `Target` provides target-dependent generators
  - targets like `X86`, `ARM`, and etc.
- `MC` manages machine code generation
- `ExecutionEngine` executes the IR
  - there are `Interpreter` and `JIT`
- `CompilerDriver` helps build gcc-like executables
- `Linker` provides linking functionality

## ExecutionEngine

- In `libLLVMInterpreter.a`, there is a static variable with constructor.  If
  it is linked, the constructor is called and the interpreter is registered to
  the `EngineBuilder`.
  - same to `libLLVMJIT.a`.
- An application uses `EngineBuilder` to create an `ExecutionEngine`.

## CodeGen

- target independent and target dependent
  - <http://llvm.org/docs/CodeGenerator.html>
  - <http://www.cs.cornell.edu/Courses/cs412/2008sp/schedule.html>
- translate LLVM IR to machine instructions
- DAG
  - Consider `%z = mul i32 %x, %y; %ans = add i32 %z, 3`
  - it is a binary tree, built bottom up
  - now, if `%ans2 = add i32 %z, %ans`
  - it is still a tree, where `%z` appears twice.
  - Or, it can be considered a DAG with `%z` having two predecessors.
  - the goal is to translate LLVM IR into a DAG
    - which is also in SSA-form
  - then use disjoint tiles to cover the DAG, where each tile can be translated
    to a machine instruction
  - the last phrase is scheduling
- `TargetLowering`
  - it is target-dependent
  - it describes the available registers and instructions of a target
  - it helps SelectionDAG lower LLVM IRs
- `SelectionDAGISel::SelectBasicBlock`
  - translate a basic block to DAG using `SelectionDAGBuilder`
  - with the help of target lowering for register using
  - the builder builds one basic block at a time
  - IRs are mapped to SelectionDAG operators
    - `ISD` for target-independent ones; `X86ISD` for target-dependent ones
  - the root node is usually the ret IR of the basic block
  - the entry node always exists, as can be seen in `SelectionDAG::clear`
- To cross  basic blocks, values need to be copy to/from virtual registers
  - after all non-terminal instructions are visited, `CopyToExportRegsIfNeeded` is called
  - if the value is used and it has virtual regs (as built in `FunctionLoweringInfo::set`),
    a copy-reg node is created.
- `getValue` returns the `SDValue` of a given `Value`
  - DAG is still in SSA form
  - if the value is the result of an instruction in the basic block, return the
    `SDValue` of the instruction
  - if the value is a constant, return the suitable constant `SDValue`
  - otherwise, it is a value from other basic blocks or function arguments
    - need to look up the register (or registers if it is array or struct)
      holding the value
    - emit `CopyFromReg` nodes for each reg and a `MERGE_VALUES` to merge them
- After `SelectionBasicBlock`, DAG is built
  - the storing registers, physical or virtual, are resolved
  - the IR ops are mapped to DAG ops
  - time to do instruction selection
- `SelectionDAGISel::CodeGenAndEmitDAG`
  - legalize and combine
