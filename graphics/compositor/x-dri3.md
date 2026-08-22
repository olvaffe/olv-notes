# X DRI3 Extension

## History

- <https://keithp.com/blogs/DRI3000/>
- released 2013
- GEM flink name is bad
  - a GEM flink name does not keep the BO alive
  - an authenticated client can access another client's BOs by guessing their
    GEM flink names
  - client and server must use the same DRM device
- server allocation is bad
  - window resize
  - cannot support `EGL_EXT_buffer_age`
- present is defined by <x-present.md>

## Model

- client allocates
- client shares using dma-bufs
  - no magic authentication anymore
  - no flink anymore
  - client can use a different GPU
- supports dma-buf-to-pixmap and pixmap-to-dma-buf
- supports pixmap present

## DRI3

- `DRI3PixmapFromBuffer`: make the prime fd the backing store of a pixmap
- `DRI3BufferFromPixmap`: get the prime fd of the backing store of a pixmap
- `DRI3FenceFromFD`: make the xshmfence fd the backing store of a server fence
- `DRI3FDFromFence`: get the xshmfence fd of the backing store of a server
                     fence
