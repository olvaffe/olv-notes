Device Tree
===========

## Basics

- <https://www.devicetree.org/>
- graph
  - the parent-child relations are not enough
  - phandles
  - ports and endpoints
- `of_platform_default_populate_init` adds `platform_device`s for OF
  `device_node`s
  - `/firmware`
  - `simple-bus`
  - `simple-mfd`
