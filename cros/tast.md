CrOS Tast
=========

## Overview

- repos
  - <https://chromium.googlesource.com/chromiumos/platform/tast/>
  - <https://chromium.googlesource.com/chromiumos/platform/tast-tests>
- `tast list <dut>`
  - `graphics.DEQP` runs a very small dEQP subset
  - `graphics.DRM` runs various `drm_tests`
  - `graphics.*`
  - `crostini.*`
  - `arc.Boot.vm`
  - `arc.DEQP.vm`
- `tast list -buildbundle crosint <dut>`

## `tast.camera.CCA*`

- `tast.camera.CCACLI`
  - setup
    - `cca setup` to update `/etc/chrome_dev.conf`
    - `/usr/local/autotest/bin/autologin.py` to login
  - `cca open` and `cca close` to open and close the camera app
  - `cca take-photo --output test.jpg` to take a photo
  - `cca record-video --duration=3 --output test.mp4` to record a video
  - `cca screenshot --output test.png` to take a screenshot
- `tast.camera.CCAUIIntent.vm`
- `tast.camera.CCAUISound`

## `tast.camera.DecodeAccelJPEG` and `tast.camera.DecodeAccelJPEGPerf`

- `DecodeAccelJPEG`
  - `restart ui`
  - `/usr/local/libexec/chrome-binary-tests/jpeg_decode_accelerator_unittest \
        --vmodule=*/media/gpu/chromeos/*=2,*/media/gpu/v4l2/*=2,*/media/gpu/vaapi/*=2,*/media/gpu/*=1 \
        --test_data_path=/usr/local/share/tast/data_pushed/go.chromium.org/tast-tests/cros/local/bundles/cros/camera/data/ \
        --jpeg_filenames=peach_pi-1280x720.jpg \
        --gtest_output=xml:/usr/local/tmp/tast/run_tmp/gtest_output.xml3726734669`
  - assets
    - <gs://chromiumos-test-assets-public/tast/cros/video/peach_pi-1280x720_20181105.jpg>
- `tast.camera.DecodeAccelJPEGPerf`
  - `restart ui`
  - `/usr/local/libexec/chrome-binary-tests/jpeg_decode_accelerator_unittest \
        --gtest_filter=All/MjpegDecodeAcceleratorTest.PerfJDA/DMABUF \
        --perf_decode_times=10000 \
        --perf_output_path=/usr/local/tmp/tast_out.20230828-164500.020854737/camera.DecodeAccelJPEGPerf/perf_output.1920x1080.hw.json \
        --test_data_path=/usr/local/share/tast/data_pushed/go.chromium.org/tast-tests/cros/local/bundles/cros/camera/data/ \
        --jpeg_filenames=pink-nature-1920x1080.jpg`
  - for sw decoding, it uses `MjpegDecodeAcceleratorTest.PerfSW`
  - assets
    - <gs://chromiumos-test-assets-public/tast/cros/video/bonsai-tree-3840x2160_20230504.jpg>
    - <gs://chromiumos-test-assets-public/tast/cros/video/peach_pi-1280x720_20181105.jpg>
    - <gs://chromiumos-test-assets-public/tast/cros/video/pink-nature-1920x1080_20230504.jpg>
    - <gs://chromiumos-test-assets-public/tast/cros/video/red-squirrel-2560x1920_20230504.jpg>
- `jpeg_decode_accelerator_unittest`'s `PerfDecodeBySW`
  - `GetSoftwareDecodeResult` calls `libyuv::ConvertToI420` to decode
    `libyuv::FOURCC_MJPG`
- `jpeg_decode_accelerator_unittest`'s `PerfDecodeByJDA/DMABUF`
  - creates a thread, `decoder_thread`
  - calls `JpegClient::CreateJpegDecoder` on the thread
  - calls `JpegClient::PrepareMemory`
    - `in_dmabuf_fd_` is a dmabuf holding the jpeg data
    - `hw_out_dmabuf_frame_`
  - calls `JpegClient::StartDecode` on the thread
    - calls `MjpegDecodeAccelerator::Decode`

## `tast.camera.HAL3Recording`

- setup
  - autologin
- `cros_camera_test --gtest_filter=Camera3ModuleFixture.NumberOfCameras:Camera3ModuleFixture.OpenDevice --3a_timeout_multiplier=2 --camera_hal_path=/usr/lib64/camera_hal/usb.so --recording_params=0:1280:720:30:1 --connect_to_camera_service=false`
- `cros_camera_test '--gtest_filter=Camera3RecordingFixture/*' --3a_timeout_multiplier=2 --camera_hal_path=/usr/lib64/camera_hal/usb.so --recording_params=0:1280:720:30:1 --connect_to_camera_service=false`

## `tast.graphics.OpenclCts.*`

- <https://chromium.googlesource.com/chromiumos/platform/tast-tests/+/refs/heads/main/src/go.chromium.org/tast-tests/cros/local/bundles/cros/graphics/opencl_cts.go>
  - this is generated
- `tast.graphics.OpenclCts.compiler_options_include_directory` executes, for
  example, `/usr/local/opencl/test_compiler options_include_directory`

## `tast.mlbenchmark.TFLite.*`

- <https://chromium.googlesource.com/chromiumos/platform/tast-tests/+/refs/heads/main/src/go.chromium.org/tast-tests/cros/local/bundles/cros/mlbenchmark/>
- test assets, `ml-test-assets.tar.gz`
  - it contains various `.tflite` files
  - it is unpacked to `/usr/local/mlbenchmark/data`
- the test runs `benchmark_model --min_secs=60 --report_peak_memory_footprint=true --graph=<path-to-tflite-file>`
  - this uses cpu
  - when the backend is `mlbenchmark.KGpuOpenCl`, `--use_gpu=true --gpu_backend=cl` are specified to use cl
- selected test outputs
  - if cpu
    - `INFO: Created TensorFlow Lite XNNPACK delegate for CPU.`
    - `VERBOSE: Replacing 58 out of 58 node(s) with delegate (TfLiteXNNPackDelegate) node, yielding 1 partitions for the whole graph.`
    - `INFO: Initialized session in 3.337ms.`
    - `INFO: count=9 first=81913 curr=55502 min=55502 max=81913 avg=58984.7 std=8115`
    - `INFO: count=1061 first=56576 curr=56299 min=54669 max=60236 avg=56265.8 std=982`
    - `INFO: Inference timings in us: Init: 3337, First inference: 81913, Warmup (avg): 58984.7, Inference (avg): 56265.8`
    - `INFO: Overall peak memory footprint (MB) via periodic monitoring: 93.2383`
  - if gpu
    - `INFO: Created TensorFlow Lite delegate for GPU.`
    - `VERBOSE: Replacing 58 out of 58 node(s) with delegate (TfLiteGpuDelegateV2) node, yielding 1 partitions for the whole graph.`
    - `INFO: Initialized OpenCL-based API.`
    - `INFO: Created 1 GPU delegate kernels.`
    - `INFO: Explicitly applied GPU delegate, and the model graph will be completely executed by the delegate.`
    - `INFO: Initialized session in 842.225ms.`
    - `INFO: count=13 first=115098 curr=36666 min=32079 max=115098 avg=40199.8 std=21694`
    - `INFO: count=1694 first=35688 curr=33419 min=30394 max=42335 avg=35134.9 std=1733`
    - `INFO: Inference timings in us: Init: 842225, First inference: 115098, Warmup (avg): 40199.8, Inference (avg): 35134.9`
    - `INFO: Overall peak memory footprint (MB) via periodic monitoring: 74.6641`
- tast parses the outputs for
  - `init_latency` is parsed from `Init:`
  - `first_inference_latency` is parsed from `First inference:`
  - `avg_latency` is parsed from `avg=`
  - `std_dev` is parsed from `std=`
  - `peak_memory` is parsed from the peak memory line
- for `convolution_benchmark_288_512_1.tflite`
  - `tflite::gpu::cl::InferenceContext::InitFromGpuModel`
    - `tflite::gpu::cl::InferenceContext::AllocateMemory` calls
      `clCreateBuffer` of size 14155776, followed by 28 `clCreateSubBuffer`
      for the buffer and 28 `clCreateImage` for the subbuffers
    - `tflite::gpu::cl::InferenceContext::Compile` calls `clCreateBuffer`,
      `clCreateImage`, `clCreateProgramWithSource`, `clBuildProgram`, and
      `clCreateKernel`
      - it creates 11 programs
      - some `ClOperation` share the same program so there are more kernels
        than programs
      - the buffers are mostly a few bytes to  kilobytes, except one that is
        221184
      - the images are 10x8, 10x16, 10x48, and 8
  - `tflite::gpu::cl::InferenceBuilderImpl::Build` calls
    `tflite::gpu::cl::InferenceRunnerImpl::LinkTensors`
    - `tflite::gpu::cl::BHWCBufferToTensorConverter::Init` creates a program
    - `tflite::gpu::cl::TensorToBHWCBufferConverter::Init` creates a program
    - `tflite::gpu::cl::AllocateTensorMemory` calls `clCreateBuffer`
      - there are two buffer of size 2359296 (`512*288*4*4`)
  - `tflite::gpu::cl::InferenceRunnerImpl::Run`
      - `CopyFromExternalObject` calls `clEnqueueWriteBuffer` and
        `clEnqueueNDRangeKernel`
      - `RunWithoutExternalBufferCopy` calls `clEnqueueNDRangeKernel` 27 times
        and `clFlush`
      - `CopyToExternalObject` calls `clEnqueueNDRangeKernel` and
        `clEnqueueReadBuffer`
      - `WaitForCompletion` calls `clFinish`

## `tast.power.*`

- `tast.power.Browsing.*`
  - `browsingTestParam`
    - `ConfigName` picks a config
      - e.g., <https://storage.googleapis.com/chromiumos-test-assets-public/power_LoadTest/v2_config/browsing_20min.json>
        - the test time is `loop_count * secs_per_page * num_page`, which is
          `2 * 60 * 10 = 20min`
    - `TimeParams` specifies metrics sampling internal (and total test time
      for sanity check)
    - `CollectTrace` enables perfetto tracing
    - `MultiTab` uses multiple tabs and windows
  - `Browsing` navigates to each page, scrolls occasionally, and moves on to
    the next page
  - variants
    - `ash` and `lacros` picks ash or lacros
    - `20min` runs for 20min rather than 1hr
    - `heavy` navigates to heavier pages
  - `power_daily`
    - `20min_ash`
    - `20min_lacros`
  - `power_regression`
    - `heavy_20min_ash`
- `tast.power.Idle.*`
  - `IdleParams`
    - `DisplayPower` controls display on/off
    - `BluetoothPower` controls bt on/off
    - `CollectTrace` enables perfetto tracing
    - `IdleTimeParams` specifies metrics sampling internal and total test time
  - `Idle` opens a blank tab and sleeps for the specified time
  - variants
    - `ash` and `lacros` picks ash or lacros
    - `fast` runs for 1min rather than 10min, with both display and bt on
  - `power_daily`
    - `display_off_bt_off_ash`
    - `display_off_bt_off_lacros`
    - `display_on_bt_on_lacros`
  - `power_regression`
    - `display_on_bt_on_ash`
- `tast.power.VideoCall.*`
  - `TimeParams` specifies metrics sampling internal and total test time
  - `VideoCall` opens two windows side-by-side
    - the first window is on
      <https://storage.googleapis.com/chromiumos-test-assets-public/power_VideoCall/power_VideoCall.webrtc.html?preset=high>
      - it uses webrtc at the top with camera and mic
      - it plays 4 video streams at the bottom
    - the second window is on <http://crospower.page.link/power_VideoCall_doc>
      - it has a text box
    - it types into the second window occasionally until the test ends
  - variants
    - `ash` and `lacros` picks ash or lacros
    - `3m`, `25m`, and `2hr` run for different total time
  - `power_daily`
    - `3m_ash`
    - `3m_lacros`
  - `power_regression`
    - `25m_ash`
- `tast.power.VideoPlayback.*`
  - `videoPlaybackTestParam`
    - `VideoName` specifies the video file
    - `TimeParams` specifies metrics sampling internal and total test time
  - `VideoPlayback` creates a fullscreen window
    - it waits until the battery is 50% charged
    - it copies the video file to `/tmp/ramdisk`
    - it plays the video file in a loop
    - it sleeps for the test time
  - variants
    - `ash` and `lacros` picks ash or lacros
    - `h264`, `vp8`, `vp9`, and `av1` are video codecs
    - `720`, `1080`, and `4k` are resolutions
    - `30fps` and `60fp` are frame rates
    - `1hr` sleeps for 1hr rather than the 6min default
  - `power_regression`
    - `h264_1080_30fps_1hr_ash`
    - `vp9_1080_30fps_1hr_ash`
  - `crosbolt_perbuild`
    - `h264_1080_30fps_ash`
    - `vp9_1080_30fps_ash`
    - `h264_1080_30fps_lacros`
    - `vp9_1080_30fps_lacros`

## `tast.power.VideoPlayback.h264_1080_30fps_1hr_ash`

- `powerAshRamfs` is the fixture
  - `NewPowerUIFixture`'s `SetUp` starts chrome and calls `PowerTestSetup`
  - `batteryDischarge.fulfill` forces using the battery
  - `setUpRamfs` mounts `ramfs` to `/tmp/ramdisk`
- `VideoPlayback` is the test function
  - it opens a blank tab and makes the browser fullscreen
  - copies `video_playback/h264_1080_30fps.mp4` to `/tmp/ramdisk`
  - cools down the dut
  - navigates to the video file
  - plays the video in a loop
  - starts the metrics recorder
  - sleeps for 1hr
  - collects the metrics
- `TestMetrics` is the data source of the metrics recorder
  - <https://chromium.googlesource.com/chromiumos/platform/tast-tests/+/refs/heads/main/src/go.chromium.org/tast-tests/cros/local/power/docs/metrics.md>
  - `NewCpuidleStateMetrics` records `CPUIdleMetricType` (`cpuidle`) metrics
    - it reads `/sys/devices/system/cpu/cpu*/cpuidle/state*/{name,latency,time}`
    - it shows how much time the cpus are in which states
  - `NewRAPLPowerMetrics` records `PowerRelatedMetricType` (`power`) metrics
    - it reads `/sys/class/powercap/intel-rapl:*/{name,energy_uj}`
      - `package-0`
        - `core`
        - `uncore`
      - `psys`
  - `NewSysfsThermalMetrics` records `ThermalMetricType` (`temperature`)
    metrics
    - it reads `/sys/class/thermal/thermal_zone*/{type,temp}`
  - `NewPackageCStatesMetrics` records `PackageCstatesMetricType` (`cpupkg`)
    metrics
    - it reads `/dev/cpu/*/msr`
  - `NewProcfsCPUMetrics` records `CPUUsageMetricType` (`cpu_usage`) metrics
    - it reads `/proc/stat` and parses the first line
  - `NewFanMetrics` records `FanMetricType` (`fan`) metrics
    - it invokes `ectool pwmgetfanrpm all` to get fan rpms
  - `NewGPUUsageDataSource` records `GPUUsageMetricType` (`gpu_usage`) and
    `GPUMemoryMetricType` (`gpu_memory`) metrics
    - it reads `/sys/kernel/debug/dri/*/clients` for client pids
    - it stats `/proc/<pid>/fd/*` for drm fds
    - it reads `/proc/<pid>/fdinfo/<fd>` for drm fdinfo
  - `NewGPUFreqMetrics` records `GPUFreqMetricType` (`gpufreq_wavg`) metrics
    - it reads
      - `/sys/kernel/debug/dri/0/i915_frequency_info` for i915
      - `/sys/kernel/debug/dri/0/amdgpu_pm_info` for amdgpu
      - `/sys/devices/platform/soc@0/5000000.gpu/devfreq/5000000.gpu/cur_freq`
        for msm
  - `NewZramIOMetrics` records `ZramMetricType` (`zram`) metrics
    - it reads `/sys/block/zram0/stat`
  - `NewMemoryMetrics` records `MemoryMetricType` (`memory)` metrics
    - it reads `/proc/meminfo`
  - `NewSysfsBatteryMetrics` records `BatterySOCMetricType` (`battery)`,
    `system`, `perf.discharge_mwh`, and `perf.minutes_battery_life_tested`
    metrics
    - it checks `/sys/class/power_supply/*/type` for `Battery`
    - it checks `/sys/class/power_supply/*/scope` for non-`Device`
    - it times `/sys/class/power_supply/*/{voltage_now,current_now}` for
      `system`
    - it reads `/sys/class/power_supply/*/charge_now` for
      `battery.battery_percent`
    - it reports total discharge in `perf.discharge_mwh`
    - it reports record time in `perf.minutes_battery_life_tested`
  - `GeneratePowerLogAndSaveToCrosbolt` derives a few more metrics
    - `perf.minutes_battery_life` projects the battery life
    - `power.non_SoC` substracts `power.package-0` from `system`

## `tast.ui.LoginPerf.ash_chrome`

- flow
  - restart ui
  - log in as cros.gaia.testing.13@gmail.com
  - opt in to play store
  - wait for arc to init
  - sign out
  - restart ui
  - repeat
    - sign in and create two windows showing animations
    - sign out
    - restart ui

## `tast.ui.MeetCUJ`

- `meetTest`
  - `bots` is an array of bot counts
    - the test seesion will be divided into N phases
    - each phase takes `duration / N` minutes and has `bots[i]` bots
  - `layout` picks meet layout (`TiledLayout` or `SpotlightLayout`)
  - `present` presents the doc or jamboard window
  - `docs` creates a doc window
  - `jamboard` creates a jamboard window
  - `split`
  - `cam` turns the camera on
  - `effects` enables meet effects
    - `Apply visual effects`, `Blur your background`
  - `backgroundBlur` enables platform bg blur
  - `adjustLighting` enables platform relighting
  - `liveCaptions` turns on platform live captioning
  - `noiseCancellation` enables platform noise cancellation
  - `zoomOut` zooms out on both the browser and display
    - `ctrl+-` and sleep 3s, for 5 times
  - `tabSwitchDocs` switches between Docs and Meet
    - `alt+tab`, sleep 10s, `alt+tab`
  - `duration` is by default 10 minutes
  - `browserType` is ash or lacros
  - `botsOptions`
- subtests
  - `tast.ui.MeetCUJ.docs`
    - `bots:        []int{1, 3, 15},`
    - `layout:      googlemeet.TiledLayout,`
    - `present:     true,`
    - `docs:        true,`
    - `split:       true,`
    - `cam:         true,`
    - `zoomOut:     true,`
    - `effects:     true,`
    - `browserType: browser.TypeAsh,`
  - `tast.ui.MeetCUJ.docs_lacros`
    - `browserType: browser.TypeLacros,`
  - `tast.ui.MeetCUJ.docs_no_effects`
    - `effects:           false,`
  - `tast.ui.MeetCUJ.docs_audio_effects`
    - `effects:           false,`
    - `liveCaptions:      true,`
    - `noiseCancellation: true,`
  - `tast.ui.MeetCUJ.docs_noise_cancellation`
    - `effects:           false,`
    - `noiseCancellation: true,`
  - `tast.ui.MeetCUJ.docs_live_captions`
    - `effects:           false,`
    - `liveCaptions:      true,`
  - `tast.ui.MeetCUJ.docs_background_blur`
    - `effects:           false,`
    - `backgroundBlur:    true,`
  - `tast.ui.MeetCUJ.docs_adjust_lighting`
    - `effects:           false,`
    - `adjustLighting:    true,`
  - `tast.ui.MeetCUJ.docs_video_effects`
    - `effects:           false,`
    - `backgroundBlur:    true,`
    - `adjustLighting:    true,`
  - `tast.ui.MeetCUJ.docs_platform_effects`
    - `effects:           false,`
    - `backgroundBlur:    true,`
    - `adjustLighting:    true,`
    - `liveCaptions:      true,`
    - `noiseCancellation: true,`
- flow
  - restart ui
  - log in as crosuicujtest1@gmail.com
  - disable play store
  - wait for cpu cooldown
  - create a window and create a meet
  - add 1 bot
  - zoom out to 50%
  - create a window and create a doc
  - show the two windows side-by-side
  - present the doc in meet
  - keep typing into doc
  - after a while, add 2 more bots
  - after a while, add 12 more bots
  - after 10m, done
- `results-chart.json` metrics
  - `Power.system` and `Power.t`
    - this shows power consumption in W at various time points in S
  - `EventLatency.KeyPressed.TotalLatency`
    - this shows keypress latentency in uS
  - `Graphics.Smoothness.PercentDroppedFrames3.AllSequences`
    - this shows frame drop rate in percentage

## `tast.video.Contents.h264_1080p_composited_hw`

- <https://chromium.googlesource.com/chromiumos/platform/tast-tests/+/refs/heads/main/src/go.chromium.org/tast-tests/cros/local/bundles/cros/video/contents.go>
  - <https://chromium.googlesource.com/chromiumos/platform/tast-tests/+/refs/heads/main/src/go.chromium.org/tast-tests/cros/local/bundles/cros/video/data/>
    provides the assets
- `h264_1080p_composited_hw`
  - `fileName` is `still-colors-1080p.h264.mp4`
  - `refFileName` is `still-colors-1080p.ref.png`
- `chromeCompositedVideo` fixture
  - it starts chrome using the args from `chromeVideoArgs`, and adds
    `--enable-hardware-overlays=""` do disable hw overlay
- `TestPlayAndScreenshot`
  - it starts an http server to serve `video.html`
  - it plays the video indefinitely fullscreen
  - it takes a screenshot after 10s

## `tast.video.PlatformDecodingPerf`

- the subtests (parameters) are generated by `platform_decoding_perf_test.go`
  - codecs are `av1`, `h264`, `hevc`, `vp8`, and `vp9`
  - resolutions are `1080` and `2160`
  - frame rates are `30` and `60`
- take `vaapi_h264_2160p_60fps` for example
  - `filename` is `perf/h264/2160p_60fps_600frames.h264`
    - there is `2160p_60fps_600frames.h264.external` pointing to
      `gs://chromiumos-test-assets-public/tast/cros/video/perf/h264/2160p_60fps_600frames_20190801.h264`
  - `decoder` is `/usr/local/libexec/chrome-binary-tests/decode_test`
  - `decoderArgsBuilder` is `H264DecodeVAAPIargsNoLogs`
    - this generates `--video=<filename> --visible --codec=H264 -v=-1`
- `measurePerformance`
  - it creates two `testexec.Cmd` with different loop args
    - `platform.LoopArgs(exec, 0, 0)` generates `--loop`
    - `platform.LoopArgs(exec, measurementIterations, framesPerIteration)`
      generates `--loop=5 --frames=200`
  - `collectMetricsPerTime`
    - it invokes the decode cmd in the background
    - `graphics.MeasureGPUCounters`
      - it measure gpu counters for `measurementDuration` (25) seconds
    - `graphics.MeasureCPUUsageAndPower`
      - it sleeps for `stabilizationDuration` (5) seconds
      - it measures cpu and power usage for `measurementDuration` (25) seconds
    - it waits for the measurement to complete
  - `collectMetricsPerRun`
    - it decodes the first `framesPerIteration` (200) frames
      `measurementIterations` (5) times
    - it measures the decode time to calculate the fps

## `tast.video.WebCodecsDecode.*`

- all tests call `webcodecs.RunDecodeTest` with different args
  - `av1_sw` plays `bear-320x240.av1.mp4` with `webcodecs.PreferSoftware`
  - `av1_hw` plays `bear-320x240.av1.mp4` with `webcodecs.PreferHardware`
  - `h264_sw` plays `bear-320x240.h264.mp4` with `webcodecs.PreferSoftware`
  - `h264_hw` plays `bear-320x240.h264.mp4` with `webcodecs.PreferHardware`
- `webcodecs.RunDecodeTest`
  - `webcodecs.prepareWebCodecsTest` starts an http server to serve
    `webcodecs_decode.html`
  - it calls `DecodeFrames` from `webcodecs_decode.js` with, for example,
    - `videoURL` is `bear-320x240.av1.mp4`
    - `width` is 320
    - `height` is 240
    - `numFrames` is 82
    - `hardwareAcceleration` is `prefer-hardware`
  - it checks the md5sums of the decoded frames against
    `bear-320x240_20211201.av1.mp4.json`
- `DecodeFrames` calls `decodeVideoInURL` to decode the video file
  - `MP4Demuxer` is the demuxer
    - it looks like a pure sw demuxer
  - `VideoDecoder` is the decoder
    - <https://developer.mozilla.org/en-US/docs/Web/API/VideoDecoder>
  - `demuxer.start` demuxes the bistream and the data is fed to the decoder
  - `decoder.decode` decodes the data

## `tast.webrtc.CaptureFromElement.*`

- there are two subtests
  - `canvas` uses `UseGlClearColor` as source and `chromeVideo` as fixture
  - `canvas_from_video` uses `UseVideo` as source and
    `chromeVideoWithFakeWebcam` as fixture
- `chromeVideoWithFakeWebcam` fixture
  - it starts chrome with special args
  - `chromeFakeWebcamArgs` uses
    - `--use-fake-device-for-media-stream`,
    - `--use-fake-ui-for-media-stream`
  - `chromeWebRTCEncodedFrameArgs` uses
    - `--enable-blink-features=RTCEncodedFrameSetMetadata,RTCEncodedVideoFrameAdditionalMetadata`
    - `--enable-features=AllowRTCEncodedVideoFrameSetMetadataAllFields`
- `RunCaptureStream`
  - starts an http server
  - navigates to
    `cros/local/bundles/cros/webrtc/data/capturefromelement.html`
  - calls js `captureFromCanvasWithVideoAndInspect`
  - early returns, because no perf measurement
- `capturefromelement.html`
  - there is a canvas element and a video element
  - `renderGetUserMediaWithPerspective`
    - `MediaDevices.getUserMedia` captures at 1270x720
    - a new video element is created to play the webcam
    - `three.js` is used to sample from the new video element and render to
      the canvas element
  - `captureFromCanvasAndInspect`
    - `canvas.captureStream` captures frames from canvas element
    - the original video element uses to the canvas element as the source
    - an `OffscreenCanvas` is created
    - `asyncIsBlackFrame` draws the original video element into the offscreen
      canvas and throws if the frames are black
