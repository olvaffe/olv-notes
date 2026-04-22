Chromium content
================

## Startup

- `chrome/app` defines the entrypoint
  - `chrome_exe_main_aura.cc` defines `main` and calls `ChromeMain`
  - `chrome_main.cc` defines `ChromeMain` and calls `ContentMain` with a
    `ChromeMainDelegate`
- `content/app` defines the app process
  - `content_main.cc` defines `ContentMain` and calls `RunContentProcess`
  - `RunContentProcess` calls `ContentMainRunner::{Initialize,Run}`
  - `content_main_runner_impl.cc` defines `ContentMainRunnerImpl::{Initialize,Run}`
    - `Initialize` calls `ChromeMainDelegate::PreSandboxStartup` which calls
      `crash_reporter::InitializeCrashpad` to fork `chrome_crashpad_handler`
    - because `process_type.empty()` is true, `Initialize` calls
      `InitializeZygoteSandboxForBrowserProcess` to fork two zygote processes
    - because `process_type.empty()` is true, `Run` calls `RunBrowser`,
      `RunBrowserProcessMain`, and `BrowserMain`
      - this process becomes the browser process
- `content/browser` defines the browser process
  - `browser_main.cc` defines `BrowserMain` and calls
    `BrowserMainRunner::{Initialize,Run}`
  - `browser_main_runner_impl.cc` defines
    `BrowserMainRunnerImpl::{Initialize,Run}`
    - `Initialize` calls `BrowserMainLoop::CreateStartupTasks` for various
      startup tasks
      - `GpuDataManagerImpl::GetInstance` initializes `GpuDataManagerImpl`
        which defines how to launch the gpu process
        - `InitializeGpuModes` initializes the possible modes
          - `gpu::GpuMode::HARDWARE_GL` is always on
          - `gpu::GpuMode::HARDWARE_VULKAN` is based on
            `ParseVulkanImplementationName`, which checks
            `features::IsUsingVulkan` (`--enable-features=Vulkan`)
      - `BrowserGpuChannelHostFactory::Initialize` establishes the gpu channel
        - `GpuProcessHost::Get` launches the gpu process by calling
          `LaunchGpuProcess`
          - `ChildProcessLauncherHelper::LaunchProcessOnLauncherThread` calls
            `ZygoteCommunication::ForkRequest` or `base::LaunchProcess` to
            fork the gpu process
          - because of `--type=gpu-process`, the gpu process's
            `ContentMainRunnerImpl::Run` calls `RunOtherNamedProcessTypeMain`
            which calls `GpuMain`
- `content/gpu` defines the gpu process
  - `gpu_main.cc` defines `GpuMain` and calls `GpuInit::InitializeAndStartSandbox`
    - stack on cros
      - `base::RunLoop::Run()`
      - `content::GpuMain()`
      - `content::RunZygote()`
      - `content::RunOtherNamedProcessTypeMain()`
      - `content::ContentMainRunnerImpl::Run()`
      - `content::RunContentProcess()`
      - `content::ContentMain()`
      - `ChromeMain`
  - `gpu/ipc` defines the `GpuInit` class
    - `service/gpu_init.cc` defines `GpuInit::InitializeAndStartSandbox`

