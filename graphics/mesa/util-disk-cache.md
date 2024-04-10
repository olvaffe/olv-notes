Mesa Disk Cache
===============

## In-Memory Cache

- apps should use a `VkPipelineCache`
  - if they don't, the driver may transparently create and use one
  - this is because different `VkGraphicsPipelineCreateInfo`s may hash and
    compile to the same shader
  - the driver does not want to waste the time
- `VkPipelineCache` is an in-memory cache
  - `sha1->pipeline` lookup
  - with reference counting, can cache pointers to pipelines than just binary
    code

## Disk Cache

- multiple levels
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
