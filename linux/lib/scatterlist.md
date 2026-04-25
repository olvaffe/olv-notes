# Kernel scatterlist

## `struct scatterlist`

- a single `struct scatterlist` describes a contiguous memory range
  - `page_link` is the starting page
    - it points to a `struct page` mostly
    - the lowest two bits are flags
      - `SG_CHAIN` indicates it points to another sg instead
      - `SG_END` indicates the last of an sg list
  - `offset` is the offset in the starting page
  - `length` is the length of the range and can exceed the starting page
  - `dma_address` is the dma address after the range has been mapped for dma
- it often exists in an array form, to describe multiple scattered ranges
  - `struct scatterlist sgl[N];`
  - `sg_init_table(sgl, N)`
    - the last entry is marked `SG_END`
  - `sg_set_page(&sgl[i], page, len, off)` for each entry
- when there are many scattered ranges, it can exist in a chain form where
  multiple arrays are chained
  - `sg_chain(sgl1, N, sgl2)`
    - `sgl1[N - 1]` must be unused and becomes `SG_CHAIN`
  - more than `SG_MAX_SINGLE_ALLOC` ranges is too many ranges
    - this is to avoid `kmalloc` needing to find contiguous pages for the sg
      array
- `for_each_sg(sgl, sg, N, i)` loops over an sg list
  - `sg` points to `sgl` on first iteration
  - `sg_next` advances `sg` on latter iterations
    - if `SG_END`, NULL
    - if `SG_CHAIN`, follow the chain
    - else, increment
- `dma_map_sg` updates `sg->dma_address` and `sg->dma_length` automatically
  - `sg->dma_address` must be updated after the pages are mapped for dma
  - `sg->dma_length` is set to `sg->length` unless iommu maps the scattered
    ranges to contiguous dma range
- dma-only sg array
  - some drm drivers can export dma-bufs for their vram
  - when `dma_buf_map_attachment` maps such a dma-buf for device dma, the sg
    entries might not have `struct page`
  - they are only suitable for device dma, and the consumer must not access
    the non-existing `sg_page`
- `CONFIG_ZONE_DEVICE` adds `ZONE_DEVICE` as a memory zone
  - `struct page` is allocated only for physical memory
  - `ZONE_DEVICE` simulates device memory (e.g., gpu vram) as hot pluggable
    physical memory
    - it hotplugs dev mem and allocs `struct page` for it
  - this allows sg to point to `struct page`, even for vram

## `struct sg_table`

- a `struct sg_table` manages an sg list
  - `sgl` is the allocated sg list
    - each array is no more than `SG_MAX_SINGLE_ALLOC` (128) entries, and they
      are chained by `sg_chain`
  - `orig_nents` is the total number of sg entries in the sg list
  - `nents` is the number of dma-mapped sg entries
- take `sg_alloc_table_from_pages` for example
  - `[offset, size]` is the data range in the discontiguous buffer described
    as a list of pages
  - to set the `sg_table` up, the number of chunks is decided first
    - when the discontiguous buffer is contiguous, the number of chunks is 1
  - `sg_alloc_table` is called with the number of chunks
    - unless there are more chunks than `SG_MAX_SINGLE_ALLOC`, it allocates an
      array of scatterlist
    - when it needs more than `SG_MAX_SINGLE_ALLOC` chunks, the last element's
      `page_link` points to the next array of scatterlist
  - `sg_set_page` is called on each scallterlist to initialize `page_link` to
    point to the starting page of the chunk
- `dma_map_sgtable` updates `sgt->nents`
  - thanks to iommu, `dma_map_sg` can map the `sgt->orig_nents` scattered
    ranges to fewer (ideally one) dma ranges for the device
  - `sgt->nents` is updated to the number of dma ranges
- `for_each_sgtable_sg` loops over every sg on `sgt->sgl`
- `for_each_sgtable_dma_sg` loops over every dma-mapped sg on `sgt->sgl`
