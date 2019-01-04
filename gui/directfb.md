DirectFB
========

## Directories

* `lib/directfb` provides `libdirect.so`
  * it is a collections of utility functions such as locking 
* `lib/fusion` provides `libfusion.so`
* `src/` provides `libdirectfb.so`
* `tools/` provides tools
* `systems/` provides `DFB_CORE_SYSTEM`s
* `wm/` provides `DFB_WINDOW_MANAGER`s
* `interfaces/` provides `DIRECT_INTERFACE_IMPLEMENTATION`s
* `inputdrivers/` provides `DFB_INPUT_DRIVER`s
* `gfxdrivers/` provides `DFB_GRAPHICS_DRIVER`s

## Important Objects

* An `IDirectFBScreen` is a screen
  * encoder, crtc, connector, and etc.
* An `IDirectFBDisplayLayer` is a hardware overlay layer
  * a layer may have a surface
* An `IDirectFBSurface` is a region of the vram.  It may be on-screen (primary)
  or off-screen.

## Init

* `DirectFBInit` allocates the global `dfb_config`, and initializes it from
  configuration files, commandline options, and etc.
* `DirectFBCreate` creates an `IDirectFB`
  * it calls `dfb_core_create` to create internal `CoreDFB` first
  * `IDirectFB_Construct` is called to construct `IDirectFB` from `CoreDFB`
* `IDirectFB::CreateSurface` creates an `IDirectFBSurface`
  * Usually, create a primary double buffered surface.  Being primary means that
    it is a scan-out (a wrapper of `/dev/fb0`)
* `IDirectFB::CreateInputEventBuffer` creates an `IDirectFBEventBuffer`

## Loop

* `IDirectFBSurface::Clear`
* `IDirectFBSurface::Flip`
* `IDirectFBEventBuffer::GetEvent` reads an event

## DirectFB/Core

* Or, `CoreDFB` created by `dfb_core_create`.  It has many parts that can be
  retrieved through `dfb_core_get_part`.  A part is defined by `DFB_CORE_PART`.
  * `dfb_clipboard_core`
  * `dfb_colorhash_core`
  * `dfb_surface_core`
  * `dfb_system_core`: a system module
  * `dfb_input_core`: an input module
  * `dfb_graphics_core`: a graphics module
  * `dfb_screen_core`
  * `dfb_layer_core`
  * `dfb_wm_core`
* It loads the system module given by `dfb_config->system`
  * by default, `FBDev`
* `DFB_CORE_SYSTEM`
  * when a system module is loaded, it calls `direct_modules_register` with
    `system_funcs`
* each part is initialized
  * for fbdev system core, it opens the fb device, optionally switches vt, and
    registers a screen with `dfb_screens_register` and a layer with
    `dfb_layers_register`
