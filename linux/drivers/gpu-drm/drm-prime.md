DRM Prime
=========

## Export Helpers

- `drm_gem_prime_handle_to_fd` is the export ioctl helper
  - `drm_gem_prime_handle_to_dmabuf`
    - if the obj is in `file_priv->prime`, `drm_prime_lookup_buf_by_handle`
      returns the dmabuf
    - if the obj is imported, it uses `obj->import_attach->dmabuf`
    - if the obj has been exported, it uses `obj->dma_buf`
    - otherwise, `export_and_register_object` exports a new dmabuf
      - `obj->funcs->export` or `drm_gem_prime_export` exports as dmabuf
        - `exp_info.resv` is `obj->resv`
        - `drm_gem_dmabuf_export`
          - `dma_buf_export` allocs the dmabuf
          - it increments obj and dev refcounts
          - `dma_buf->file->f_mapping` is set to `dev->anon_inode->i_mapping`
      - `obj->dma_buf` is set to the singleton dmabuf
    - `drm_prime_add_buf_handle` tracks the dmabuf in `file_priv->prime`
  - `fd_install` points an fd to the dmabuf
- `struct dma_buf_ops` helpers
  - `drm_gem_map_attach` is called when an importer attaches
    - `obj->funcs->pin` pins the obj to keep the pages around
  - `drm_gem_map_dma_buf` is called when an importer maps to get sgt for hw dma
    - `obj->funcs->get_sg_table` returns sgt for the pinned obj pages
      - `drm_prime_pages_to_sg` is a helper
    - `dma_map_sgtable` maps the sgt for hw dma (e.g., iommu setup)
  - `drm_gem_unmap_dma_buf` is called when an importer unmaps
    - `dma_unmap_sgtable` unmaps the sgt for hw dma (e.g., iommu teardown)
    - it frees the sgt
  - `drm_gem_map_detach` is called when an importer detaches
    - `obj->funcs->unpin` unpins obj pages
  - `drm_gem_dmabuf_mmap` is called when a dmabuf is mmaped to userspace
    - this happens when the userspace mmaps the dmabuf fd, or when the
      importer supports mmap of an imported dmabuf
    - `drm_gem_prime_mmap` maps the obj for userspace vma
      - `vma->vm_ops` is set to `obj->funcs->vm_ops`
      - `obj->funcs->mmap` preps the vma
        - if the obj is imported, this chains to the original exporter
        - otherwise, it preps vma flags (wc, etc.) and lets `vm_ops` handle
          faults
  - `drm_gem_dmabuf_vmap` is called when a dmabuf is vmapped for kernel
    - this is rarely used
    - `obj->funcs->vmap` vmaps the obj
  - `drm_gem_dmabuf_vunmap` is called when a dmabuf is vunmapped from kernel
  - `drm_gem_dmabuf_release` is called when the last ref of dmabuf is dropped
    - it drops the refcounts of the obj and the dev

## Import Helpers

- `drm_gem_prime_fd_to_handle` is the import ioctl helper
  - if the obj is in `file_priv->prime`, it is nop
    - this ensures a unique handle for the dmabuf
  - `dev->driver->gem_prime_import` or `drm_gem_prime_import` imports as obj
    - if the dmabuf was exported by us, `drm_gem_is_prime_exported_dma_buf`
      returns true and it returns `dma_buf->priv` directly
    - `dma_buf_attach` attaches dmabuf
    - `dma_buf_map_attachment_unlocked` maps dmabuf and returns sgt
    - `dev->driver->gem_prime_import_sg_table` imports sgt as obj
      - this is driver-specific
    - `obj->import_attach` is initialized
    - `obj->resv` points to `dma_buf->resv`
  - `drm_prime_add_buf_handle` tracks the dmabuf in `file_priv->prime`
- `drm_prime_gem_destroy` destroys an imported obj
  - `dma_buf_unmap_attachment_unlocked` unmaps dmabuf
  - `dma_buf_detach` detaches dmabuf
- `drm_prime_get_contiguous_size` returns contiguous size of an sgt
  - the hw might require contiguous dma range, after iommu-mapped if any, to
    import
