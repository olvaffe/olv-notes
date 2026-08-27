# Android libgraphicsenv

## Java Framework

- gpu layer injection
  - this does not require root, but the app must be debuggable
    - or has `com.android.graphics.injectLayers.enable`
  - settings
    - `enable_gpu_debug_layers` is the master switch
    - `gpu_debug_app` is the debuggable app
    - `gpu_debug_layers` is the colon-seprated list of vk layers
    - `gpu_debug_layers_gles` is the colon-seprated list of gles layers
    - `gpu_debug_layer_app` is the app that provides the layer
  - activitiy main thread
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
- angle
  - settings
    - `angle_gl_driver_all_angle` forces all apps to load angle
    - `angle_gl_driver_selection_pkgs` and `angle_gl_driver_selection_values`
      force listed app to load the specified driver
    - `angle_dynamic_denylist` forces listed app to load the native driver
    - `angle_debug_package` is the app that provides the angle driver
    - `angle_egl_features` is forwarded into angle
  - `GraphicsEnvironment.java`
    - `getDriverForPackage`
      - if `ANGLE_GL_DRIVER_ALL_ANGLE`, return `angle`
      - else check `ANGLE_GL_DRIVER_SELECTION_PKGS` and
        `ANGLE_GL_DRIVER_SELECTION_VALUES`
    - `shouldUseAngle` returns true when `getDriverForPackage` returns `angle`
    - `getAngleDebugPackage` returns `ANGLE_DEBUG_PACKAGE`
    - `setAngleInfo` passes angle info to native code

## Native `GraphicsEnv`

- `GraphicsEnv`
  - `setDriverPathAndSphalLibraries` is called when updated driver is used
    - `getDriverNamespace` returns null typically unless updated driver is used
  - `setDriverToLoad`, `setVulkanInstanceExtensions`, etc. are called from
    egl and vulkan loaders to update gpu stats
  - `setLayerPaths` and `setDebugLayers*` are called from java to set up app
    debugging layers, if any
