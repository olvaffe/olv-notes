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

## Unigine Valley

- `Unigine_Valley-1.0`
- run the benchmark and `ps -ef` shows
    ./valley_x64 \
        -project_name Valley \
        -data_path ../ \
        -engine_config ../data/valley_1.0.cfg \
        -system_script valley/unigine.cpp \
        -sound_app openal \
	-video_app opengl \
	-video_multisample 0 \
	-video_fullscreen 1 \
	-video_mode -1 \
	-video_height 720 \
	-video_width 1280 \
	-extern_define ,RELEASE,LANGUAGE_EN,QUALITY_HIGH \
	-extern_plugin ,GPUMonitor

## Vulkan Samples

- build
  - `git clone --recurse-submodules https://github.com/KhronosGroup/Vulkan-Samples.git`
  - `cd Vulkan-Samples`
  - `cmake -H. -Bout -GNinja -DCMAKE_BUILD_TYPE=Debug`
  - `ninja -C out`
  - `ln -sf out/app/bin/Debug/x86_64/vulkan_samples`
- main loop
  - stack
    - `DescriptorManagement::update`
    - `vkb::Application::step`
      - app here is a `DescriptorManagement`
    - `vkb::VulkanSamples::update`
    - `vkb::Application::step`
      - app here is a `VulkanSamples` (not `VulkanSample`)
    - `vkb::Platform::run`
    - `vkb::Platform::main_loop`
  - when the window is out of focus and is not in benchmark mode,
    `vkb::Platform::run` returns immediately and the test will be in a busy
    loop
- swapchain
  - stack of image acquire
    - `vkb::Swapchain::acquire_next_image`
    - `vkb::RenderContext::begin_frame`
    - `vkb::RenderContext::begin`
    - `DescriptorManagement::update`
  - stack of image present
    - `vkb::Queue::present`
    - `vkb::RenderContext::end_frame`
    - `vkb::RenderContext::submit`
    - `vkb::RenderContext::submit`
    - `DescriptorManagement::update`

## GravityMark

- <https://gravitymark.tellusim.com/>
  - Android
  - Linux x86-64 and arm64

## glmark2

- <https://github.com/glmark2/glmark2>
  - `./src/glmark2-es2-wayland --data-path ~/projects/glmark2/data -b build:model=bunny`
  - `-l` to see all scenes (e.g., `build`) and scene options (e.g., `model`)
- `MainLoop::step`
  - `do_benchmark` enters the mainloop and keeps calling `step`
  - on scene begin,
    - `before_scene_setup` deletes `fps_renderer_` and `title_renderer_`
    - `CanvasGeneric::reset`
      - reset gl state
      - create a new window
      - create a new context and make current
      - ensure an fbo if offscreen
      - `glViewport`, `glClear`, etc.
    - `Benchmark::setup_scene` loads resources, builds vbos, etc.
    - `after_scene_setup` checks if `show-fps` and `title` options are set
      - they display title and fps on screen
    - `log_scene_info` prints scene info to stdout
  - `MainLoop::draw`
    - `CanvasGeneric::clear` to clear
    - `scene_->draw` to render
    - `scene_->update` to update state
    - `CanvasGeneric::update` to swap or `glFinish`
  - on scene end,
    - `Benchmark::teardown_scene` to unload resources, etc.
    - `Scene::average_fps` calculates the fps which is added to `score_`
      - fps is calculated as `currentFrame_ / realTime_.elapsed()`
    - `log_scene_result` prints fps, etc to stdout
