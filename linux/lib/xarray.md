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
  - `xa->xa_head` points to the node of the current highest level
    - if the highest index is less than 2^6, level 1
    - if the highest index is less than 2^12, level 2
    - ...
    - if the highest index is less than 2^60, level 10
    - that is, for small indices, the tree is no more than one or two levels
- `xas_alloc` allocs a new `xa_node`
  - `node->shift` is the specified shift
  - `node->count` is the number of children nodes
  - `node->parent` is the parent node (level N+1)
  - `node->array` points back to the array
- `xas_expand` grows the tree vertically as needed
  - if node `head` does not cover index `max`, `xas_alloc` allocs a node in
    the next level and updates `xa->xa_head` to point to the new node
  - updates `xas->xa_node` to point to the new highest node for the index
- `xas_create` ensures necessary nodes for an index are allocated
  - given the index, `slot` is the slot at level N and `entry` is the node at N-1
    - if `entry` is NULL, `xas_alloc` allocs a new node and updates `slot`
    - `xas_descend` returns the node at N-2
      - `xas->xa_node` is set to the node at N-1
      - `xas->xa_offset` is set to the offset at level N-1
- `xa_store(xa, idx, obj, 0)` is like `xa[idx] = obj`
  - `xas_create` allocs all nodes for the index; when it returns,
    - `xas->xa_node` is the parent node for the index
    - `xas->xa_offset` is the slot offset for the index
    - ret is the old value, `xas->xa_node->slots[xas->xa_offset]`

