Chromium `third_party/blink`
============================

## GPU and WebGL

- WebKit expects the renderer to create a `WebKitClient`
- WebKit calls `WebKitClient::createGLES2Context` to create a `WebGLES2Context`
- the renderer implements `WebGLES2Context` with `ggl`, which in turns uses
  `gpu` for indirect rendering (renderer process to gpu process)
- the renderer sends commands to the gpu process (`chrome_gpu`)
  - the commands are handled by `GPUProcessor`
  - decoded GL commands are dispatched to `app/gfx/gl` directly
  - context management commands are sent to who?
- remote gles
  - initially, remote gles was used for everything
    - renderer/ui/display compositors all used remote gles, but they use
      remote raster now after OOP-D / OOP-R
    - webgl used remote gles and still does today
  - the client uses `GLES2Implementation`
    - it implements `GLES2Interface`, which includes
      `gles2_interface_autogen.h` to declare all GLES2 functions
    - it includes `gles2_implementation_autogen.h` and
      `gles2_implementation_impl_autogen.h` to implement all GLES2 functions
    - it uses `GLES2CmdHelper` to serialize the GLES2 function calls into a
      `CommandBufferProxyImpl`
  - the service in the gpu process uses `GLES2DecoderImpl` or
    `GLES2DecoderPassthroughImpl`
    - it uses `GLES2CommandBufferStub` to deserialize the commands and
      dispatches to `GLES2DecoderImpl::HandleFoo` or
      `GLES2DecoderPassthroughImpl::HandleFoo`
    - `GLES2DecoderImpl` validates before calling into the driver while
      `GLES2DecoderPassthroughImpl` calls into angle directly
- `SharedImageBacking`
  - webgl renders to a `SharedImageBacking` and compositor samples from the
    `SharedImageBacking`
  - by default on linux/x11, the image backing is `GLTextureImageBacking`
    - that is, webgl renders to a GL texture and compositor samples from it
    - `--enable-features=DefaultANGLEVulkan` uses `GLTextureImageBacking`
      - that is, everything works the same except angle translates gles to vk
        internally
    - `--enable-features=Vulkan` uses `ExternalVkImageBacking`
      - webgl renders to an imported vk image
      - compositor samples from the vk image
      - it crashes on radv and runs fine on anv
    - `--enable-features=Vulkan,DefaultANGLEVulkan` uses `OzoneImageBacking`
      - webgl renders to an imported gbm bo
      - compositor samples from the imported gbm bo
      - it gets tiling wrong on radv and runs fine on anv
    - `--enable-features=Vulkan,DefaultANGLEVulkan,VulkanFromANGLE` uses
      `AngleVulkanImageBacking`
      - webgl renders to a vk image
      - compositor samples from the vk image
