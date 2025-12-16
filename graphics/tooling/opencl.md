OpenCL Apps
===========

## Loaders

- `libOpenCL.so.1` is provided by
  - <https://github.com/KhronosGroup/OpenCL-ICD-Loader>, or more commonly,
  - <https://github.com/OCL-dev/ocl-icd>
- `OpenCL-ICD-Loader`
  - `OCL_ICD_ENABLE_TRACE=1` enables loader debug messages
  - `OCL_ICD_FILENAMES` specifies addtional ICDs that are `dlopen`ed before
    scanning the vendors path
  - `OCL_ICD_VENDORS` specifies an alternative vendors path rather than the
    default `/etc/OpenCL/vendors`
    - each `.icd` file under the vendors path contains a path that is `dlopen`ed
  - `OPENCL_LAYER_PATH` specifies an alternative layers path rather than the
    default `/etc/OpenCL/layers`
    - each `.lay` file under the layers path contains a path that is `dlopen`ed
    - all layers are implicitly loaded?
  - `OPENCL_LAYERS` specifies additional layers that are `dlopen`ed after
    scanning the layers path
- `ocl-icd`
  - `OCL_ICD_DEBUG=15` enables loader debug messages
    - `1` for warnings
    - `2` for infos
    - `4` for function enter/leave
    - `8` dumps icd info
  - `OCL_ICD_VENDORS` specifies
    - the vendors path to be scanned,
    - the `.icd` file to be parsed, or
    - the icd library to be `dlopen`ed

## Layers

- Intercept Layer for OpenCL
  - build
    - `git clone https://github.com/intel/opencl-intercept-layer.git`
    - `cmake -S . -B out -G Ninja`
  - use
    - it is not a CL layer that can be loaded via `OPENCL_LAYERS`
    - it is to be used as the CL library for apps, and it will chain load the
      real CL library
    - `LD_LIBRARY_PATH=<path-to-intercept-layer>` such that app finds the
      inteceptor
    - `CLI_OpenCLFileName=<path-to-real-cl-impl>` tells the interceptor to
      dlopen the specified cl impl
  - config `~/clintercept.conf`
    - env `CLI_<option>` sets the same option
    - Setup and Loading Controls
      - `OpenCLFileName` (string)
        - the path of the real icd loader
    - Logging Controls
      - `LogToFile` (bool)
        - log to `clintercept_log.txt` instead of stderr
      - `CallLogging` (bool)
        - log all cl calls
      - `CallLoggingThreadId` or `CallLoggingThreadNumber` (bool)
        - log thread id or number
      - `CallLoggingElapsedTime` (bool)
        - log call time
      - `ErrorLogging` (bool)
        - log cl errors
      - `FlushFiles` (bool)
        - flush after each log write
      - `DumpDir` (string)
        - default to `~/CLIntercept_Dump/<Process Name>`
    - Reporting Controls
    - Controls for Dumping and Injecting Programs and Build Options
      - `DumpProgramSource` (bool)
        - dump all program source
      - `DumpProgramBinaries` (bool)
        - dump all program binaries
      - `AppendBuildOptions` (string)
        - append build options
      - `AppendLinkOptions` (string)
        - append link options
    - Controls for Emulating Features
    - Controls for Automatically Creating SPIR-V Modules
    - Controls for Dumping Buffers and Images
    - Device Partitioning Controls
    - Capture and Replay Controls
    - AubCapture Controls
    - Execution Controls
    - Platform and Device Query Overrides
    - Precompiled Kernel and Builtin Kernel Override Controls
- OpenCL Kernel Profiler
  - build
    - `git clone https://github.com/rjodinchr/opencl-kernel-profiler.git`
    - `cmake -S . -B out -G Ninja -DPERFETTO_SDK_PATH=~/projects/perfetto/sdk -DOPENCL_HEADER_PATH=/usr/include`
  - use
    - `OPENCL_LAYERS=<path>/libopencl-kernel-profiler.so CLKP_TRACE_MAX_SIZE=100000 prog` will generate
      `opencl-kernel-profiler.trace`
      - without `CLKP_TRACE_MAX_SIZE`, the trace file is truncated after 1MB
