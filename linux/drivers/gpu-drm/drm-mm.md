DRM MM
======

## Overview

- `drm_mm` is a memory manager
  - `dev->vma_offset_manager` uses `drm_mm` to manage mmap offsets
  - each gpu vm has a `drm_mm`
- `drm_mm_init` inits a `drm_mm`
- `drm_mm_takedown` cleans up a `drm_mm`
  - it warns if the mm is not empty
- `drm_mm_insert_node_in_range` allocs a range
- `drm_mm_remove_node` removes a range
- `drm_mm_reserve_node` reserves a range whose begin/end are externally
  determined
- `drm_mm_scan_*` is legacy mechanism for eviction
