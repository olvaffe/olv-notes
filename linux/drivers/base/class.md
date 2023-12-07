Kernel Driver Class
===================

## Initialization

- `classes_init` is called from `driver_init` during driver subsystem init
- `class_kset` is created with `kset_create_and_add` with no parent kobject
  - it appears in `/sys/class`

## Class Registration

- `class_register` registers a new `class` under `/sys/class`
  - a `subsys_private` is allocted to represent the class
    - `cp->class` points to the specified `class`
    - `cp->subsys` is the kset of the class (`/sys/class/<foo>`)
    - it is also used for buses
      - unlike `bus_register`, `cp->devices_kset` and `cp->drivers_kset` are
        not used
- `class_create` allocs a `class` and calls `class_register` on it
- `class_interface_register` registers a new `class_interface` to a class
  - it is added to `sp->interfaces`
  - `class_interface` has `add_dev` and `remove_dev` callbacks that are
    called when devices are added to and removed from the class
