kobject
=======

## kernfs

- a `kernfs_node` has
  - `parent` pointing to the parent node
  - if the node is a dir, `kernfs_elem_dir` with `children` and `root`
  - if the node is a file, `kernfs_elem_attr`
  - if the node is a symlink, `kernfs_elem_symlink`
- `kernfs_create_root` creates a root `kernfs_node` and a `kernfs_root`
  pointing to the root node
- `kernfs_create_dir` creates a dir node
- `kernfs_create_file` createa a file node
- `kernfs_create_link` creates a symlink node
- this gives us a nice tree structure

## kobject

- from `kobject_init_and_add`, a kobject
  - optionally has an parent
  - optionally belongs to a kset
    - a kset is another kobject and a list of children kobjects
  - if no parent specified, kset becomes the parent
  - if no parent nor kset, there will be no parent
  - a directory `kernfs_node`
- kobjects do not form a tree; their `kernfs_node`s do

## sysfs

- each kobject has a dir `kernfs_node` created by `sysfs_create_dir_ns`
  - if the kobject has a parent, the new node is under the parent's node
  - otherwise, the new node is under the root node
- `sysfs_create_file` creates a file `kernfs_node` under the kobject's dir
  `kernfs_node`
- `sysfs_create_group` creates a dir `kernfs_node` under the kobject's dir
  `kernfs_node`
