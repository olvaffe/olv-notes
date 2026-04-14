Android libgraphicsenv
======================

## `GraphicsEnv`

- activity main thread
  - `ActivityThread::setupGraphicsSupport` calls `GraphicsEnvironment::setup`
    - `setupGpuLayers`
      - `setLayerPaths_native` calls `GraphicsEnv::setLayerPaths` to add
        search paths (e.g., app dir for bundled layers)
      - if debuggable, and `enable_gpu_debug_layers` and `gpu_debug_app`
        settings are set, `setDebugLayers_native` calls
        `GraphicsEnv::setDebugLayers` and `setDebugLayersGLES_native` calls
        `GraphicsEnv::setDebugLayersGLES`
    - `setupAngle` choses native or angle gles driver
    - `chooseDriver` chooses built-in or updated driver
- `GraphicsEnv`
  - `setDriverPathAndSphalLibraries` is called when updated driver is used
    - `getDriverNamespace` returns null typically unless updated driver is used
  - `setDriverToLoad`, `setVulkanInstanceExtensions`, etc. are called from
    egl and vulkan loaders to update gpu stats
  - `setLayerPaths` and `setDebugLayers*` are called from java to set up app
    debugging layers, if any

## ANGLE

- `GraphicsEnvironment.java`
  - `getDriverForPackage`
    - if `ANGLE_GL_DRIVER_ALL_ANGLE`, return `angle`
    - else check `ANGLE_GL_DRIVER_SELECTION_PKGS` and
      `ANGLE_GL_DRIVER_SELECTION_VALUES`
  - `shouldUseAngle` returns true when `getDriverForPackage` returns `angle`
  - `getAngleDebugPackage` returns `ANGLE_DEBUG_PACKAGE`
  - `setAngleInfo` passes angle info to native code
