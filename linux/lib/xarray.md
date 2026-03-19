Kernel xarray
=============

## Overview

- an `xarray` is an associative array that maps indices to pointers
  - it is a modern implementation of radix tree
- radix tree is deprecated
  - `#define radix_tree_root xarray`
  - `#define radix_tree_node xa_node`
- idr is deprecated
  - it wraps a `radix_tree_root`
- ida is still in-use
  - it wraps an `xarray`

## Radix Tree

- an `xarray` is a radix tree
  - the 64-bit index is divided into 6-bit (`XA_CHUNK_SHIFT`) chunks
    - level 0: bit 0-5, leaves
    - level 1: bit 6-11
    - ...
    - level 10: bit 60-63, root
  - each `xa_node` has 64 slots
    - level 0: shift 0
    - level 1: shift 6
    - ...
    - level 10: shift 60
- `xas_alloc` allocs a new `xa_node`
  - `node->shift` is the specified shift
  - `node->count` is the number of children nodes
  - `node->parent` is the parent node (level N+1)
  - `node->array` points back to the array
- `xas_expand` grows the tree vertically
  - if node `head` does not cover index `max`, `xas_alloc` allocs a node in
    the next level and updates `xa->xa_head` to point to the new node
    - that is, for small indices, the tree is no more than one or two levles
  - updates `xas->xa_node` to point to the new highest node for the index

- `xa_store(xa, idx, obj, 0)` is like `xa[idx] = obj`
  - `xas_create` creates a slot for storage
    - 
