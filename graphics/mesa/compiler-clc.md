Mesa CLC
========

## OpenCL Compiler

- an CL implementation may or may not include a compiler
- when it includes a compiler, it is typically LLVM-based
  - clang compiles the OpenCL C source code to LLVM IR
    - e.g., `-cl-std=CL3.0 -cl-ext=+cl_khr_fp64`
    - clang provides `opencl-c.h` to declare OpenCL C types/functions
  - llvm links in OpenCL C functions
    - libclc is an implementation of OpenCL C functions
    - they are implemented in OpenCL C and are pre-compiled to LLVM bytecode
  - the backend compiler compiles LLVM IR to binary
- libclc
  - take `mad` for example
    - `generic/include/clc/math/mad.h` declares `mad` variants
    - `generic/lib/math/mad.cl` implements `mad` variants in OpenCL
      - it lowers `mad` to internal `__clc_mad`
    - `clc/include/clc/math/clc_mad.h` declares internal `__clc_mad`
    - `clc/lib/generic/math/clc_mad.cl` implements internal `__clc_mad`
      - it returns `a * b + c` with `FP_CONTRACT ON`
  - take `sin` for example
    - it lowers `sin` to its taylor series
  - it invokes clang to compile `*.cl` to bytecode
    - if the target is spirv, it invokes `llvm-spirv` to convert `.bc` to
      `.spv`

## Headers

- `nir_clc_helpers.h` loads libclc and translates it to nir
  - `nir_can_find_libclc` has no user
  - `nir_load_libclc_shader`
    - `open_clc_data` loads libclc spirv, or uses embedded ones
    - `spirv_to_nir` translates spirv to nir
    - `vtn_handle_opencl_instruction` calls `mangle_and_find` to find the
      function from the libclc nir
- `clc_helpers.h`
  - these wrap llvm, clang, and llvm-spirv
    - `clc_initialize_llvm` inits llvm
    - `clc_c_to_spir` uses clang to compile opencl c to llvm ir
      - spir is llvm ir pinned down to a certain version
    - `clc_spir_to_spirv` uses llvm-spirv to translate llvm ir to spirv
    - `clc_c_to_spirv` uses clang and llvm-spirv to compile opencl c to spirv
  - these wrap spirv-tools
    - `clc_spirv_get_kernels_info` calls `spvBinaryParse` to parse spirv
      binary
    - `clc_free_kernels_info`
    - `clc_link_spirv_binaries` calls `spvtools::Link` to link multiple spirv
      binaries
    - `clc_spirv_tools_version` calls `spvSoftwareVersionString`
    - `clc_validate_spirv` calls `spvtools::SpirvTools::Validate`
    - `clc_spirv_specialize` uses
      `spvtools::CreateSetSpecConstantDefaultValuePass` to specialize
    - `clc_dump_spirv` calls `spvtools::SpirvTools::Disassemble`
    - `clc_free_spir_binary`
    - `clc_free_spirv_binary`
- `clc.h` provides wrappers
  - `clc_libclc_new` is a wrapper to `nir_load_libclc_shader`
  - `clc_free_libclc`
  - `clc_libclc_get_clc_shader`
  - `clc_libclc_serialize`
  - `clc_libclc_free_serialized`
  - `clc_libclc_deserialize`
  - `clc_compile_c_to_spir` is a wrapper to `clc_c_to_spir`
  - `clc_free_spir`
  - `clc_compile_spir_to_spirv` is a wrapper to `clc_spir_to_spirv`
  - `clc_free_spirv`
  - `clc_compile_c_to_spirv` is a wrapper to `clc_c_to_spirv`
  - `clc_link_spirv` is a wrapper to `clc_link_spirv_binaries`
  - `clc_parse_spirv` is a wrapper to `clc_spirv_get_kernels_info`
  - `clc_free_parsed_spirv` is a wrapper to `clc_free_kernels_info`
  - `clc_specialize_spirv` is a wrapper to `clc_spirv_specialize`
  - `clc_debug_flags` inits debug flags from `CLC_DEBUG`
