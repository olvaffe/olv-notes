Kernel and devres
=================

## Overview

- `devres_alloc` allocs a device resource
  - the device resource has an internal type, `devres`
    - `node` is the resource header
      - `entry` is the list node
      - `release` is the release callback
      - `name` is the function name of the release callback
      - `size` is the user data size
    - `data` is the caller data
  - only `dr->data` is returned
- the caller inits device resource
  - only `dr->data` that is caller-defined
- `devres_add` adds the device resource
  - it adds `dr` to `dev->devres_head`
- when the device is destroyed, or unbound from its driver,
  `devres_release_all` releases all resources
