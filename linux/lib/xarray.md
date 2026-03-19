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
