Linux VFS Super Block
=====================

## Overview

- a `super_block` represents a filesystem
  - for each physical fs, there is exactly one `super_block`
  - two mounts can share the same super block
  - it is destroyed when the last mount goes away
