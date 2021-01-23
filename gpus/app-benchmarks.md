GPU Benchmarks
==============

## Basemark GPU

- basemarkgpu-1.2.3
- run the custom benchmark and `ps -ef` shows
    ./resources/binaries/BasemarkGPU_vk \
        TestType Custom \
        TextureCompression bc7 \
        RenderPipeline simple \
        RenderResolution 1280x720 \
        LoopCount 1 \
        GpuIndex 0 \
        ProgressBar true \
        AssetPath ./resources/assets/pkg \
        StoragePath ./logs \
        SkipZPrepass true
- `RenderPipeline` can be `simple`, `medium`, or `highend`

## Unigine Heaven

- `Unigine_Heaven-4.0`
- run the benchmark and `ps -ef` shows
    ./heaven_x64 \
        -project_name Heaven \
        -data_path ../ \
        -engine_config ../data/heaven_4.0.cfg \
        -system_script heaven/unigine.cpp \
        -sound_app openal \
        -video_app opengl \
        -video_multisample 0 \
        -video_fullscreen 0 \
        -video_mode 3 \
        -extern_define RELEASE,LANGUAGE_EN,QUALITY_HIGH,TESSELLATION_DISABLED \
        -extern_plugin GPUMonitor
  - `cat /proc/<pid>/environ | tr '\0' '\n'` shows
    - `LD_LIBRARY_PATH=x64`
- windows version works under wine
