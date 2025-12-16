SDL
===

## Build

- `git clone https://github.com/libsdl-org/SDL.git`
  - it is SDL3 by default
- `cmake -S . -B out -G Ninja -DCMAKE_BUILD_TYPE=Debug -DCMAKE_INSTALL_PREFIX=$(pwd)/out/install`
- `ninja -C out`

## Internals of Basics

- using wayland and vulkan
- Initialization
  - `SDL_Init`
    - `SDL_EventsInit`
    - `SDL_VideoInit` calls `Wayland_VideoInit`
      - it binds to these globals
        - `wl_compositor`
        - `wl_output`
        - `wl_seat`
        - `xdg_wm_base`
        - `wl_shm`
        - `zwp_relative_pointer_manager_v1`
        - `zwp_pointer_constraints_v1`
        - `zwp_keyboard_shortcuts_inhibit_manager_v1`
        - `zwp_idle_inhibit_manager_v1`
        - `xdg_activation_v1`
        - `zwp_text_input_manager_v3`
        - `wl_data_device_manager`
        - `zwp_primary_selection_device_manager_v1`
        - `zxdg_decoration_manager_v1`
        - `zwp_tablet_manager_v2`
        - `zxdg_output_manager_v1`
        - `wp_viewporter`
  - `SDL_CreateWindow`
    - `Wayland_CreateWindow`
      - `wl_compositor_create_surface`
    - `SDL_FinishWindowCreation`
      - `SDL_ShowWindow` unless `SDL_WINDOW_HIDDEN` is set
        - `Wayland_ShowWindow`
          - `xdg_wm_base_get_xdg_surface`
          - `xdg_surface_get_toplevel`
        - `SDL_SendWindowEvent(SDL_WINDOWEVENT_SHOWN)`
  - `SDL_Vulkan_GetInstanceExtensions` calls
    `Wayland_Vulkan_GetInstanceExtensions`
  - `SDL_Vulkan_CreateSurface` calls `Wayland_Vulkan_CreateSurface`
    - `vkCreateWaylandSurfaceKHR`
- Event Handling
  - `SDL_WaitEvent`
    - `Wayland_PumpEvents`
      - `WAYLAND_wl_display_flush` flushes pending requests to the socket
      - `WAYLAND_wl_display_prepare_read` and `WAYLAND_wl_display_read_events`
        reads events from the socket
      - `WAYLAND_wl_display_dispatch_pending` dispatches events that
        have been read from the socket
        - this calls various callbacks registered with `wl_*_add_listener`
  - `SDL_PollEvent`
- Window State
  - `SDL_MinimizeWindow` calls `Wayland_MinimizeWindow`
    - `xdg_toplevel_set_minimized`
    - note that minimized is not a state of `xdg_shell` 
  - `SDL_MaximizeWindow` calls `Wayland_MaximizeWindow`
    - `xdg_toplevel_set_maximized`
    - it also sets `SDL_WINDOW_MAXIMIZED`
  - `SDL_RestoreWindow` calls `Wayland_RestoreWindow`
    - `xdg_toplevel_unset_maximized`
    - it also unsets `SDL_WINDOW_MAXIMIZED`
  - `SDL_SetWindowFullscreen` calls `Wayland_SetWindowFullscreen`
    - `xdg_toplevel_set_min_size`
    - `xdg_toplevel_set_max_size`
    - `xdg_toplevel_set_fullscreen`
    - `xdg_toplevel_unset_fullscreen`
    - `WAYLAND_wl_display_roundtrip` forces a roundtrip
- Window Events
  - `handle_configure_xdg_shell_surface` handles `xdg_surface::configure`
    - this send `SDL_SendWindowEvent(SDL_WINDOWEVENT_RESIZED)`
