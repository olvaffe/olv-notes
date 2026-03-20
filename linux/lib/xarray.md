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
    - level 0: bit 0-5
    - level 1: bit 6-11
    - ...
    - level 10: bit 60-63
  - the 64-bit entry (value) is interpreted as followed
    - if the lowest two bits are 00, a pointer aligned to 4 bytes
    - if the lowest two bits are 10, an internal entry
      - node, sibling, retry, or zero
    - if the lowest one bit is 1, a value
  - each `xa_node` has 64 slots
    - level 0: shift 0
    - level 1: shift 6
    - ...
    - level 10: shift 60
  - `xa->xa_head` points to the current tree top
    - if the highest index is less than 2^6, first node at level 0
    - if the highest index is less than 2^12, first node at level 1
    - ...
    - if the highest index is less than 2^66, first node at level 10
    - that is, for small indices, the tree height is like 1 or 2
  - to traverse the tree, starting from the current tree top at `xa->xa_head`,
    - `node = &node->slots[(index >> node->shift) & 0x3f]` descends one level
- `xa_state` is a temp variable used during tree traversal
  - `xas->xa` is the array
  - `xas->xa_index` is the index
  - when valid, `xas->xa_node->slots[xas->xa_offset]` is the current slot/entry
  - otherwise, `xas->xa_node` can have special values
    - `XA_ERROR` indicates an error
    - `XAS_BOUNDS` indicates out-of-bound
    - `XAS_RESTART` indicates to restart from `xa->xa_head`
- `xas_load` traverses the tree and returns the entry at the specified index
  - it conceptually returns `xa[idx]`
  - `xas_start` starts the traversal
    - `xas->xa_node` is set to NULL
    - `xa->xa_head`, the current tree top, is returned
  - each iteration calls `xas_descend` to descend one level
    - `xas->xa_node` is set to the parent node
    - `xas->xa_offset` is set to the slot offset in the parent node
    - `xas->xa_node->slots[xas->xa_offset]`, the current entry, is returned
- `xas_create` is similar to `xas_load`, except it creates all missing nodes
  - `xas_expand` grows the tree height if needed
    - if `head` does not cover `max`, that is, if the current tree top does
      not cover the specified index, it allocates a new node for the next
      level and updates `xa->xa_head` to point to the new node
    - `xas->xa_node` is set to the current tree top
    - it returns the shift of the next level
  - each iteration calls `xas_descend` to descend one level
    - `entry` is the parent node or NULL
    - `slot` is the slot for the parent node in the grandparent node
    - `node` is the parent node
      - if missing, `xas_alloc` allocs a new one and `*slot = node` to update
        the slot in the grandparent node
    - `xas_descend` descends one level
      - `xas->xa_node` is set to the parent node
      - `xas->xa_offset` is set to the slot offset in the parent node
      - `xas->xa_node->slots[xas->xa_offset]`, the current entry, is returned
    - `slot` is updated
- `xa_store` traverses the tree and stores the entry at the specified index
  - it conceptually does `xa[idx] = entry`
  - `xas_create` traverses to the slot and returns the current entry
    - the slot is `&xas->xa_node->slots[xas->xa_offset]`
  - the rest does `*slot = entry` to update the slot, but is complex due to
    `CONFIG_XARRAY_MULTI`
