Android librenderengine
=======================

## Public Headers

- `librenderengine` is a static library
  - surfaceflinger is the only consumer
- `renderengine/DisplaySettings.h`
  - `struct DisplaySettings` describes settings a display that affect gpu
    composition
  - `drawLayers` takes a `DisplaySettings`
- `renderengine/ExternalTexture.h` and `renderengine/impl/ExternalTexture.h`
  - `class ExternalTexture` is an abstract class of the output buffer
  - `drawLayers` takes an `ExternalTexture`
- `renderengine/LayerSettings.h`
  - `struct LayerSettings` describes settings a layer that affect gpu
    composition
  - `drawLayers` takes a vector of `LayerSettings`
- `renderengine/RenderEngine.h`
  - `class RenderEngine` is a render engine
  - `create` creates a render engine from `RenderEngineCreationArgs`
    - `RenderEngineCreationArgs::Builder` builds a `RenderEngineCreationArgs`
  - `dump` dumps states for dumpsys
  - `supportsProtectedContent` returns if re supports protected content
  - `onActiveDisplaySizeChanged` notifies display size change
  - `drawLayers` draws the layers to the buffer
  - `tonemapAndDrawGainmap` tonemaps an hdr buffer to an sdr buffer for
    screenshot
  - `cleanupPostRender` is called after `drawLayers`
  - `validateInputBufferUsage` asserts `GRALLOC_USAGE_HW_TEXTURE`
  - `validateOutputBufferUsage` asserts `GRALLOC_USAGE_HW_RENDER`
