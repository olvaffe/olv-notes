GPU Performance Analysis
========================

## GPU Top

- <https://github.com/rib/gputop>

## GPUVis

- <https://github.com/mikesart/gpuvis>

## gfx-pps

- <https://gitlab.freedesktop.org/Fahien/gfx-pps>

## Vulkan Shader Profiler

- build perfetto
  - `git clone https://android.googlesource.com/platform/external/perfetto`
  - `tools/install-build-deps`
  - `tools/gn args out`
    - `is_clang = true`
    - `is_debug = false`
    - `cc_wrapper = "ccache"`
  - `ninja -C out libtrace_processor.a`
- build spirv-tools
  - `git clone https://github.com/KhronosGroup/SPIRV-Tools`
  - `python3 utils/git-sync-deps`
  - `cmake -S. -Bout -GNinja`
- build vsp
  - `git clone https://github.com/rjodinchr/vulkan-shader-profiler`
  - `cmake -S. -Bout -GNinja -DBACKEND=System \
       -DSPIRV_TOOLS_SOURCE_PATH=$HOME/projects/SPIRV-Tools \
       -DSPIRV_TOOLS_BUILD_PATH=$HOME/projects/SPIRV-Tools/out \
       -DSPIRV_HEADERS_INCLUDE_PATH=$HOME/projects/SPIRV-Tools/external/spirv-headers/include \
       -DPERFETTO_SDK_PATH=$HOME/projects/perfetto/sdk \
       -DPERFETTO_TRACE_PROCESSOR_LIB=$HOME/projects/perfetto/out/libtrace_processor.a \
       -DPERFETTO_INTERNAL_INCLUDE_PATH=$HOME/projects/perfetto/include \
       -DPERFETTO_GEN_INCLUDE_PATH=$HOME/projects/perfetto/out/gen/build_config \
       -DPERFETTO_CXX_CONFIG_INCLUDE_PATH=$HOME/projects/perfetto/buildtools/libcxx_config \
       -DPERFETTO_CXX_SYSTEM_INCLUDE_PATH=$HOME/projects/perfetto/buildtools/libcxx/include \
       -DEXTRACTOR_NOSTDINCXX=1 \
       -DCMAKE_CXX_COMPILER=clang++`
    - it seems, because vsp uses perfetto's internal headers, it has to use
      perfetto's internal copy of libcxx and clang
