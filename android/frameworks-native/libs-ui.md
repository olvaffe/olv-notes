Android libui
=============

## Graphics

- `GraphicBufferMapper` is a singleton
  - it loads the mapper hal in-process
  - `get` returns the singleton
    - the ctor tries these mapper hal wrappers in-order
      - `Gralloc5Mapper`
      - `Gralloc4Mapper`
      - `Gralloc3Mapper`
      - `Gralloc2Mapper`
  - `importBuffer` imports a `native_handle_t` and returns a `buffer_handle_t`
    - it is a wrapper for `mMapper->importBuffer`
  - `freeBuffer` frees an imported buffer
    - it is a wrapper for `mMapper->freeBuffer`
  - `lock*` locks a buffer for cpu access
    - they are wrappers for `mMapper->lock`
  - `unlock*` unlocks a buffer
    - they are wrappers for `mMapper->unlock`
  - `get*` queries buffer metadata
    - they are wrappers for `mMapper->get*`
- `GraphicBufferAllocator` is a singleton
  - it talks to the allocator and is privileged
  - `get` returns the singleton
    - the ctor gets the mapper singleton first
    - it uses the allocator wrapper that matches the mapper
      - `Gralloc5Allocator`
      - `Gralloc4Allocator`
      - `Gralloc3Allocator`
      - `Gralloc2Allocator`
  - `allocate` allocates a `buffer_handle_t`
    - it is a wrapper for `mAllocator->allocate`
  - `free` frees an allocated buffer
    - it is a wrapper for `mMapper.freeBuffer`
- `GraphicBuffer`
  - the ctor gets the mapper singleton
    - when allocating, the ctor calls `initWithSize`
      - this gets the allocator singleton to allocate
    - when importing, the ctor calls `initWithHandle` to import
      - this calls `mBufferMapper.importBuffer`
    - there is also `unflatten` that also imports
  - the dtor calls `mBufferMapper.freeBuffer` or `allocator.free`
  - it is a subclass of `ANativeWindowBuffer`
  - it is a reinterpret-cast to `AHardwareBuffer`

## Gralloc5

- `GraphicBufferMapper` is implemented on top of `Gralloc5Mapper` (or an older
  version)
  - `getInstance` returns the singleton `Gralloc5`, which provides both the
    allocator and the mapper
    - `waitForAllocator` waits for the allocator service to be up
    - `loadIMapperLibrary` loads the mapper hal
      - `android_load_sphal_library` loads `mapper.<vendor>.so`
    - it calls the mapper hal's `AIMapper_loadIMapper` to get the `AIMapper`
      interface
  - most methods are thin wrappers to the mapper hal's `AIMapper` interface
- `GraphicBufferAllocator` is implemented on top of both `Gralloc5Mapper` and
  `Gralloc5Allocator` (or an older version)
  - `getInstance` returns the singleton `Gralloc5`
  - `allocate` is a wrapper for `mAllocator->allocate2`
    - `importBuffer` is usually true and `mMapper.importBuffer` is called to
      import the buffer
    - `importBuffer` can be false when the buffer is allocated but not used by
      the calling process

## minigbm

- gralloc5
  - there is only generic support using dumb bo
  - allocator
    - `Android.bp` builds `android.hardware.graphics.allocator-service.minigbm`
    - `allocator.rc` invokes the binary when `vendor.graphics.allocator` is
      started
    - `allocator.xml` is the aidl manifest for the allocator
  - mapper
    - `Android.bp` builds `mapper.minigbm`
    - `mapper.minigbm.xml` is the aidl manifest for the mapper
- gralloc4 intel
  - allocator
    - `Android.bp` builds `android.hardware.graphics.allocator@4.0-service.minigbm_intel`
    - `android.hardware.graphics.allocator@4.0-service.minigbm_intel.rc` invokes
      the binary when `vendor.graphics.allocator-4-0` is started
    - `android.hardware.graphics.allocator@4.0.xml` is the hidl manifest for
      the allocator
  - mapper
    - `Android.bp` builds `android.hardware.graphics.mapper@4.0-impl.minigbm_intel`
    - `android.hardware.graphics.mapper@4.0.xml` is the hidl manifest for the
      allocator
- gralloc0 intel
  - `Android.bp` builds `gralloc.minigbm_intel`
