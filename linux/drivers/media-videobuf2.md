V4L videobuf2
=============

## `struct vb2_mem_ops`

- `alloc` allocates the storage, or `attach_dmabuf` to use another dma-buf as
  the storage
- `vaddr` maps the buffer for kernel access
- `mmap` maps the buffer for userspace access
- before and after device access, `prepare` and `finish` should be called
  respectively

## `videobuf2-dma-sg`

- From its `struct vb2_mem_ops`, `vb2_dma_sg_memops`, we can see that
  - `alloc` calls `kvmalloc_array` and calls `dma_map_sg_attrs` to map the
    pages for DMA
  - `prepare` and `finish` call `dma_sync_sg_for_device` and
    `dma_sync_sg_for_cpu` respectively
- From its `struct dma_buf_ops`, `vb2_dma_sg_dmabuf_ops`, we can see that
  - there is no `begin_cpu_access` and `end_cpu_access`
  - there is no `dma_fence` nor `dma_resv`
  - v4l userspace must synchronize GPU and media access to the buffer
