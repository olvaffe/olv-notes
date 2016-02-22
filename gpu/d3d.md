Direct3D
========

## DXGI

* Since Direct3D 10, the window system dependent part is moved to DXGI
* `IDXGIFactory`
  * A DXGI factory is created by `CreateDXGIFactory(__uuidof(IDXGIFactory), &ptr);`
  * `factory->CreateSoftwareAdapter` creates an software `IDXGIAdapter`
  * `factory->EnumAdapters` creates an `IDXGIAdapter`
  * `factory->CreateSwapChain` creates an `IDXGISwapChain`
    * a swap chain is created with a `IDXGIDevice`, which is a base class for a
      D3D device
    * `D3D11CreateDevice` creates a `pipe_context` from the `pipe_screen` of an
      adaptor and wraps them in a d3d device
* `IDXGIAdapter` and `IDXGIOutput`
  * `adapter->EnumOutputs` creates an `IDXGIOutput`
  * wraps a `native_display` and supports modesetting
* `IDXGIDevice` and `IDXGISurface`
  * An `IDXGIDevice` is implemented in the D3D layer because the `CreateSurface`
    method creates a `IDXGISurface` that is used by D3D.
  * `IDXGISurface` provides `Map` and `Unmap` for CPU access
* `IDXGISwapChain`
  * is created with a `IDXGIDevice`.  That is, a `pipe_context`.
  * is also created with an `desc.OutputWindow`.  It is used to create a
    `native_surface`.
  * `swapchain->GetBuffer` asks the dxgi device to create a surface
    (`pipe_resource`) and return the surface to the caller (back buffer)
  * `swapchain->Present` blits the buffer from `GetBuffer` to the front or back
    buffer of `native_surface` and calls `flush_front` or `swap_buffers`
* Swap Chain
  * a swap chain has one front buffer and one or more back buffers
  * a swap chain is created in the video memory for fast presentation
  * when a window is resized, the app should
    * `g_pd3dDevice->OMSetRenderTargets(0, 0, 0);`
    * `g_pRenderTargetView->Release();`
    * `g_pSwapChain->ResizeBuffers(...);`
    * `g_pSwapChain->GetBuffer(...)`
    * `g_pd3dDevice->CreateRenderTargetView(...)`
    * `g_pd3dDevice->OMSetRenderTargets(...)`
    * That is, release references to swap chain buffer, resize, and acquire the
      new buffer.
* Gallium
  * `IDXGIFactory` is a wrapper to the X11 display and `native_platform`
  * `IDXGIAdapter` creates a `native_display`
    * `IDXGIOutput` is for modesetting

## Windows Vista Display Driver Model

* <http://msdn.microsoft.com/en-us/library/ff569513%28v=VS.85%29.aspx>
* Paired display user-mode driver and kernel-mode display driver
* The kernel-mode display driver is called miniport driver
* There is D3D runtime that loads the user-mode driver
  * the user-mode driver implements DXGI DDI interface and D3D10 and/or D3D11
    DDI interface
  * runtime calls `OpenAdater10` of the user-mode driver to open an instance of
    the graphics adapter.  It passes `D3D10DDIARG_OPENADAPTER` to the user-mode
    driver.  The user-mode driver uses the data structure to query the miniport
    driver capabilities
  * The adapter has `D3D10DDI_ADAPTERFUNCS` which contain function pointers
    implemented by the user-mode driver for `CreateDevice` and others.  They
    are called by the runtime.
  * A device has `D3D10DDIARG_CREATEDEVICE` which contain DXGI DDI and State DDI
    function pointers implemented by the user-mode driver.  They are called by
    the runtime.
  * On the other hand, the user-mode driver calls into the runtime
    * the runtime passes pointers to `D3D10DDI_CORELAYER_DEVICECALLBACKS` and
      `DXGI_DDI_BASE_CALLBACKS` to the user-mode driver when it creates a
      device.
* The miniport driver has a single access point `DriverEntry`.
  * `DriverEntry` initializes `DRIVER_INITIALIZATION_DATA` with function
    pointers to functions implemented by the miniport
  * it then calls `DxgkInitialize` provided by `Dxgkrnl.sys`
  * at a later point, `Dxgkrnl.sys` passes `DXGKRNL_INTERFACE` to the miniport
    driver which is also a list of function pointers that the miniport driver
    can call.
* OpenGL ICD
  * very similar to a user-mode driver in that it is a user-space driver and it
    calls into the kernel-mode driver.  There is also OpenGL runtime.
  * instead of callback structures created by the runtime, OpenGL ICD loads
    `gdi32.dll` and calls `GetProcAddress` to get the address of `D3DKMT*`
    functions.   They can be used to access `Dxgkrnl.sys`
  * the runtime, `opengl32.dll`, loads the ICD
    * all core `wgl*` calls from app to the runtime are translated into `Drv*`
      calls in the ICD.  WGL and GL are paired.  In Intel driver, `Drv*` will
      call into the GL.
    * all core `gl*` calls from app to the runtime are translated to the same
      `gl*` calls in the ICD

## Direct3D 11

* create a swap chain, device, and device context
  * use the device for resource/object creation; use the context for rendering
* ask the swap chain for the back buffer
* ask the device to create a render target view backed by the buffer
* ask the device context to bind the view as the render target
* set the viewport
