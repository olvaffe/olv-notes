Mesa util
=========

## `getenv` wrapper

- `getenv`
- `os_get_option`
- `debug_get_*option`
- `DEBUG_GET_ONCE_*OPTION`

## Disk Cache

- In-Memory Cache
  - apps should use a `VkPipelineCache`
    - if they don't, the driver may transparently create and use one
    - this is because different `VkGraphicsPipelineCreateInfo`s may hash and
      compile to the same shader
    - the driver does not want to waste the time
  - `VkPipelineCache` is an in-memory cache
    - `sha1->pipeline` lookup
    - with reference counting, can cache pointers to pipelines than just binary
      code
- Disk Cache
  - gallium drivers create a `disk_cache` to cache hw binaries
  - st reusues the cache for NIR
  - glsl compiler reuses the cache for linked GLSL IR
    - it also reuses the cache to defer compiling GLSL to GLSL IR.  The idea
      is that when there is a cache hit for the linked GLSL IR, it does not
      need to compile GLSL.  Otherwise, it does a deferred compilation
  - it is simpler for vulkan drivers
- API
  - `disk_cache_compute_key` computes the 20-byte disk cache key from a
    driver-specific shader key
  - `disk_cache_remove` is used by the glsl compiler to remove corrupted
    entries
  - `disk_cache_has_key` and `disk_cache_put_key` are optimized versions of
    get/put, and are used by the glsl compiler
  - `disk_cache_set_callbacks` is used by dri frontend to implement
    `EGL_ANDROID_blob_cache`
  - `disk_cache_get` loads the data from disk
  - `disk_cache_put` adds a job to write the data to disk
    - `cache_item_metadata` is for GLSL metdata, and is designed for Steam
  - `disk_cache_put_nocopy` is the same as above, except the data is not
    copied (and must be alive until `disk_cache_wait_for_idle`)
  - `disk_cache_format_hex_id` formats a byte array to hex-string
  - `disk_cache_get_function_timestamp` returns the mtime of the library
    containing the symbol
  - `disk_cache_get_function_identifier` adds `note.gnu.build-id` to the sha1
    context, or falls back to use `disk_cache_get_function_timestamp`

## driconf

- driver first defines a `driOptionDescription` array for available options
  - with helpers such as `DRI_CONF_SECTION` and `DRI_CONF_SECTION_END`
- driver calls `driParseOptionInfo` to parse the available options
  - this parses the `driOptionDescription` array into a `driOptionCache`
- driver calls `driParseConfigFiles` to apply the config
  - `screenNum` is almost always 0
    - only `dri2` sets it
  - `driverName` is the driver name
  - `kernelDriverName` is almost always NULL
    - only `loader` sets it
  - `deviceName` is almost always NULL
    - only `msm` sets it to, for example, `FD630` with the help of
      `fd_dev_name`
  - `applicationName` is vk app name
  - `applicationVersion` is vk app version
  - `engineName` is vk engine name
  - `engineVersion` is vk engine version
- drivers that use driconf
  - vulkan: `anv`, `dzn`, `radv`, `turnip`, and `venus`
    - anv and hasvk both use the name `anv`
  - gallium: `crocus`, `iris`, `msm`, `radeonsi`, `v3d`, `virtio_gpu`, and
    `zink`
  - misc: `loader`, `nine`, `dri2`, wgl, pipe-loader
    - wgl uses the pipe driver name determined in `wgl_screen_create`
    - pipe-loader uses the DRI driver name
- DRI drivers
  - when EGL gets the fd from the display server, it calls
    `loader_get_user_preferred_fd` to get the preferred fd
    - this is mainly for prime
    - the device selection can be done via `DRI_PRIME` or
      -`<device driver="loader">` and `<option name="device_id" value="..."/>`
  - EGL then calls `loader_get_driver_for_fd` to get the DRI driver name
    - the driver selection can be done via `MESA_LOADER_DRIVER_OVERRIDE` or
      -`<device driver="loader">` and `<option name="dri_driver" value="..."/>`
  - DRI drivers consist of
    - the dri2 frontend, which understands a few more options
      - `glx_extension_override`
      - `indirect_gl_extension_override`
      - `vblank_mode`
      - `block_on_depleted_buffers`
    - the pipe loader, whic understands many options
      - `driinfo_gallium.h` is common
      - `driinfo_iris.h` and etc are driver-specific
  - EGL uses `__DRI2configQueryExtension` to access dri2's options
  - EGL uses `__DRIconfigOptionsExtension` to implement
    `EGL_MESA_query_driver`
    - this returns the merged XML string, which can be used by GUI app
- `driParseConfigFiles`
  - `driParseConfigFiles` also uses `execname` is from
      - `MESA_DRICONF_EXECUTABLE_OVERRIDE`
      - `MESA_PROCESS_NAME`
      - `program_invocation_name`
  - sample xml
  
      <driconf>
        <device>
          <application>
            <option/>
          </application>
          <engine>
            <option/>
          </engine>
        </device>
      </driconf>
  - `optConfStartElem` and `optConfEndElem` are the element handlers
  - when `<device>` is parsed,
    - `parseDeviceAttr` parses the attributes
    - only one of these attributes are used, in order of priority
      - `driver`
      - `kernel_driver`
      - `device`
      - `screen`
    - the attribute is used for device maching
      - `ignoringDevice` is set if not matched and the entire element is skipped
  - when `<application>` is parsed,
    - `parseAppAttr` parses the attributes
    - only one of the attributes are used for matching
      - `name` attr is ignored
