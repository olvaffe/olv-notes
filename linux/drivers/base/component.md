Kernel and component
====================

## Overview

- component helper is typically used across multiple drivers
  - there is a master driver and one or more component drivers
  - component drivers call `component_add` in their probe functions
  - master driver calls these functions in its probe function
    - `component_match_add` adds a match for a component
    - `component_master_add_with_match` adds the master
  - when all components become available, master driver calls
    `component_bind_all`
- `component_add`
  - it allocs a `component` and adds it to global `component_list`
  - `try_to_bring_up_masters` calls `try_to_bring_up_aggregate_device` on all
    incomplete masters, and if a master becomes complete thanks to the
    new component, it calls the master's bind
- `component_match_add`
  - it allocs a `component_match`, which is a dynamic array internally, on demand
  - the match data is saved to the dynamic array
- `component_master_add_with_match`
  - it allocs a `aggregate_device` and adds it to global `aggregate_devices`
  - `try_to_bring_up_aggregate_device` checks if a master is complete, and
    calls its bind
- `component_bind_all`
  - it calls all components' bind
- IOW,
  - master driver and component drivers probe their devices in any order
    - they add the devices to the component helper without really probing them
  - when all devices are probled, the component helper calls the master's bind
    automatically
  - the master's bind calls `component_bind_all` to call components' bind
