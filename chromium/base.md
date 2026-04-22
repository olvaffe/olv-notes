Chromium Base
=============

## Switches, Features, and Flags

- switches
  - switches are defined everywhere
    - `find -name '*_switches.cc'` for most of them
  - switches are parsed everwhere
    - `base::CommandLine::ForCurrentProcess()->HasSwitch(switches::kFooBar)`
    - `base::CommandLine::ForCurrentProcess()->GetSwitchValue(switches::kFooBar)`
- features
  - features are defined everywhere
    - `find -name '*_features.cc'` for most of them
    - they are defined by `BASE_FEATURE`
      - there are more than 4K features
      - each feature has a name and is enabled/disabled by default
    - `--enable-features=Foo,Bar` is just another switch parsed by
      `FeatureList::InitializeInstance`
      - `FeatureList::RegisterOverride` is called on each enabled feature with
        `OVERRIDE_ENABLE_FEATURE` to update `overrides_`
  - `FeatureList::IsEnabled(features:kFoo)`
    - this calls `GetOverrideStateByFeatureName` to loop over `overrides_`
    - `OVERRIDE_ENABLE_FEATURE` overrides to enabled
    - `OVERRIDE_DISABLE_FEATURE` overrides to disabled
    - `OVERRIDE_USE_DEFAULT` uses the feature's `default_state`
      - `FEATURE_ENABLED_BY_DEFAULT` defaults to enabled
      - `FEATURE_DISABLED_BY_DEFAULT` defaults to disabled
  - <https://chromium.googlesource.com/chromium/src/+/main/testing/variations/>
    - `VariationsFieldTrialCreatorBase::SetUpFieldTrials` sets up field trials
    - on a non-chrome dev build, `FIELDTRIAL_TESTING_ENABLED` is set by default
      - `VariationsFieldTrialCreatorBase::ApplyFieldTrialTestingConfig`
        applies the testing config generated from
        `fieldtrial_testing_config.json`
- flags
  - flags are defined in `chrome/browser/about_flags.cc`
  - each flag is associated with a switch or a feature
    - it provides a means to set switches/features persistently
    - not all switches/features have associated flags
- gpu
  - <https://chromium.googlesource.com/chromium/src/+/main/docs/gpu/debugging_gpu_related_code.md>
  - `--disable-gpu-sandbox`
  - `--no-sandbox`
  - `--enable-features=Vulkan,DefaultANGLEVulkan,VulkanFromANGLE`
    - `Vulkan` uses skia-vk rather than skia-gl
    - `DefaultANGLEVulkan` usea angle-vk rather than angle-gl (passthrough)
    - `VulkanFromANGLE` makes skia-vk and angle-vk share VkInstance and
      VkDevice
  - `--disable-gpu-process-crash-limit`
- logging
  - `--enable-logging=stderr`
    - `DetermineLoggingDestination`
    - logging is enabled by default only on debug builds
  - `--log-level=0`
    - `InitChromeLogging`
    - `0` is `LOGGING_INFO` and is the lowest severity
    - `LOG(severity)` is enabled when `LOG_IS_ON(severity)` returns true
      - `--log-level=0` to enable `LOGGING_INFO`
  - `--v=1`
    - `VlogInfoFromCommandLine`
    - `VLOG(verbosity)` is enabled when `VLOG_IS_ON(verbosity)` returns true
      - `--v=1` to enable verbosity 1
  - `--disable-logging-redirect`
    - `RedirectChromeLogging`
    - on cros, this is the default (set by the session manager) on a test build
    - otherwise, logging is redirected to `/home/chronos/user/log/chrome`
      after user login
  - `DLOG` and `DVLOG` are enabled when `DCHECK_ALWAYS_ON` is enabled at
    compile time and the respective sevrity/verbosity is enabled at runtime

## Threads

- <https://chromium.googlesource.com/chromium/src/+/HEAD/docs/threading_and_tasks.md>
- <https://chromium.googlesource.com/chromium/src/+/HEAD/docs/callback.md>
- browser process
  - the main thread ends up in `BrowserMainLoop::RunMainMessageLoop`, running
    `RunLoop`
  - the io thread is created by `BrowserMainLoop::CreateThreads`
  - the thread pool is created by `StartBrowserThreadPool`
- gpu process
  - the main thread ends up in `GpuMain`, running `RunLoop`
  - the io thread is created by `ChildProcess`
  - the thread pool is also created by `ChildProcess`
- task posting
  - a task is essentially a function call wrapped by `base::BindOnce`
  - `base::ThreadPool::CreateSingleThreadTaskRunner` or
    `base::SingleThreadTaskRunner::GetCurrentDefault` returns a single thread
    task runner
    - tasks posted to the single thread task runner are executed in-order on
      the same thread from the thread pool
  - `base::ThreadPool::CreateSequencedTaskRunner` or
    `base::SequencedTaskRunner::GetCurrentDefault` returns a sequenced task
    runner
    - tasks posted to the sequenced task runner are executed in-order, but may
      be on different threads from the thread pool
    - a "sequence" is a virtual thread
  - `base::ThreadPool::PostTask` posts a task to the thread pool
    - two tasks may be executed out-of-order
- physical threads
  - `base::SimpleThread` is an abstract class representing a physical thread
    - `SimpleThread::Start` calls `PlatformThread::CreateWithType` to create a
      physical thread
    - the physical thread runs `base::SimpleThread::ThreadMain` which calls
      `base::SimpleThread::Run` virtual function
  - `base::Thread` is a concrete class representing a physical thread
    - `Thread::Start` calls `PlatformThread::CreateWithType` to create a
      physical thread
    - the physical thread runs `base::Thread::ThreadMain` which enters
      `RunLoop::Run` to handle tasks

## Perfetto

- Chrome uses TrackEvent data source in a different way
  - use perfetto ui, `Record new trace`, `Chrome` to record a trace
  - then go to `Info and stats` to see the exact config used
    - instead of `track_event` data source and `track_event_config`, it uses

      data_sources: {
        config {
          name: "org.chromium.trace_event"
          chrome_config {
              trace_config: "..."
          }
        }
      }
- <https://source.chromium.org/chromium/chromium/src/+/main:base/trace_event/builtin_categories.h>
  - `cc` is the compositor
    - <https://source.chromium.org/chromium/chromium/src/+/main:cc/>
    - in the renderer process, blink is the client
    - in the browser process, ui is the client
  - `disabled-by-default-skia.gpu` is skia gpu (ganesh and graphite)
    - <https://source.chromium.org/chromium/chromium/src/+/main:third_party/skia/src/gpu/>
  - `evdev` is ozone evdev (only used on cros)
    - <https://source.chromium.org/chromium/chromium/src/+/main:ui/events/ozone/evdev/>
  - `drm` is ozone drm (only used on cros)
    - <https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/drm/>
    - `hwoverlays` for overlay-related events
    - `drmcursor` for cursor-related events
  - `exo` is exo wayland compositor (only used on cros)
    - <https://source.chromium.org/chromium/chromium/src/+/main:components/exo/>
  - `input` is input
    - <https://source.chromium.org/chromium/chromium/src/+/main:content/browser/renderer_host/input/>
  - `mojom` is mojo ipc
    - <https://source.chromium.org/chromium/chromium/src/+/main:mojo/>
  - `gpu` is gpu-related events
    - <https://source.chromium.org/chromium/chromium/src/+/main:gpu/>
  - `gpu.angle` is angle
    - <https://source.chromium.org/chromium/chromium/src/+/main:third_party/angle/>
  - `ozone` is ozone abstraction layer
    - <https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/>
  - `toplevel` is top-level events
  - `toplevel.flow` is top-level flow events
  - `ui` is ui
    - <https://source.chromium.org/chromium/chromium/src/+/main:ui/>
  - `v8` is v8 js engine
    - <https://source.chromium.org/chromium/v8/v8>
  - `views` is ui views toolkit
    - <https://source.chromium.org/chromium/chromium/src/+/main:ui/views/>
  - `viz` is for composition and gpu presentation
    - <https://source.chromium.org/chromium/chromium/src/+/main:components/viz/>
  - `wayland` is ozone wayland
    - <https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/>
- `EnablePerfettoSystemTracing`
  - this feature is enabled by defaut for cros
  - `PerfettoTracedProcess::SetupSystemTracing` connects to the system
    perfetto socket
  - the gpu process cannot connect to the perfetto socket due to sandboxing,
    which can be disabled by `--disable-gpu-sandbox`
