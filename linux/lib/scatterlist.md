Kernel scatterlist
==================

## Overview

- A `struct scatterlist` describes a contiguous memory range
  - `page_link` is the starting page
  - `offset` is the offset in the starting page
  - `length` is the length of the range and can exceed the starting page
- A `struct sg_table` manages `struct scatterlist` nodes
  - `sg_alloc_table` allocates the specified count of nodes
  - when the number of nodes exceeds `SG_MAX_SINGLE_ALLOC` (128),
    `sg_alloc_table` makes multiple memory allocations for the nodes.
    - A `struct scatterlist` can also act as a pointer to another array of
      scatter list.  See `sg_chain`.
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
