Kernel kobject
==============

## `struct kref`

- an `atomic_t` is a typedef'ed struct with one member, `int`
  - use `atomic_*` such as `atomic_inc` to access an `atomic_t`
- a `refcount_t` is a typedef'ed struct with one member, `atomic_t`
  - use `refcount_*` such as `refcount_inc` to access a `refcount_t`
  - it prints warnings when the refcount saturates or underflows
  - it uses relaxed memory ordering
- a `kref` is a struct with one member, `refcount_t`
  - use `kref_*` such as `kref_get` to access a `kref`
  - it has a few `kref_put_*` variants for ease of use

## `struct kobject`

- <https://docs.kernel.org/core-api/kobject.html>
- a `kobject` has these members
  - `ktype` is the mandatory type of the object
  - `name` is the mandatory name owned by the object
  - `sd` is the mandatory sysfs dir node owned by the object
  - `parent` is the optional parent object
  - `kset` is the optional set the object belongs to
  - `entry` is to link the object to a `kset`
  - `kref` is for refcount
  - some bool flags
- initialization
  - `kobject_init` initializes a kobject embedded in a larger struct
    - it requires a `kobj_type`
    - `state_initialized` is set
  - `kobject_add` adds a kobject to the hierarchy
    - `kobject_set_name_vargs` allocates and sets the name
    - `parent` is optional
    - if `kobj->set` is non-null, the object is added to the set
      - if there is no parent, the set becomes the parent
    - `sysfs_create_dir_ns` allocates and sets `kernfs_node` for the object
      - this creates a dir node
      - if there is no parent, the dir node is under `sysfs_root_kn`
    - `state_in_sysfs` is set
  - `kobject_init_and_add` combines `kobject_init` and `kobject_add`
  - `kobject_create_and_add` allocs and adds a kobj dynamically
    - `ktype` is set to `dynamic_kobj_ktype`
- destruction
  - `kobject_del` undoes `kobject_add`
  - `kobject_put` releases a kobj if the refcount reaches 0
    - `kobject_cleanup` takes care of the flow
    - `__kobject_del` is called to undo `kboject_add`
    - `kobj_type::release` is called to relesae the object
    - `name` is freed
- operations
  - `kobject_rename` renames a kobj
    - e.g., net interface rename
    - this involves renaming the sysfs dir node, sending a `KOBJ_MOVE` uevent,
      etc.
  - `kobject_move` reparents a kobj
    - this involves reparenting the sysfs dir node, the kobj, and sending a
      `KOBJ_MOVE` uevent
  - `kobject_get_path` walks the kobj hierarchy and returns a path suitable
    for `DEVPATH` (e.g., `/bus/foo/bar`)
  - `kobject_uevent_env` builds an uevent and sends it to userspace via
    `NETLINK_KOBJECT_UEVENT` netlink socket

## `struct kset`

- a `kset` has these members
  - `list` is the list of kobjs in the set
  - `list_lock` is a spinlock
  - `kobj` because the set is itself a kobj
  - `uevent_ops` gives the set a chance to modify uevents (e.g., add
    `SUBSYSTEM`)
- initialization
  - `kset_init` initializes a set without adding it to the hierarchy
    - this is rarely used externally
  - `kset_register` calls `kset_init`, `kobject_add_internal`, and
    `kobject_uevent`
    - this is rarely used externally
    - the caller must have filled out `kset->kobj.ktype`, etc.
  - `kset_create_and_add` calls `kset_create` and `kset_register`
    - `kset_create` allocs a `kset`, initializes `kset->kobj` with
      `kset_ktype`
- destruction
  - `kset_unregister` calls `kobject_del` to undo `kobject_add`
  - `kset_put` calls `kobject_put`
- operations
  - `kobject_add` adds a kobj to a set if `kobj->set` is set
