Kernel cgroup
=============

## Overview

- `cgroup.c`
  - `struct cgroup_root` represents a cgroup hierarchy
    - there is a single hierarchy, `cgrp_dfl_root`, in v2
  - `struct cgroup` represents a cgroup
    - there is a `struct cgroup` embedded in `struct cgroup_root`
  - `struct css_set` represents the state of a cgroup
    - there is 1:1 between `struct cgroup` and `struct css_set` in v2
    - `find_css_set` allocs `css_set` on demand
    - `init_css_set.dfl_cgrp` points to `cgrp_dfl_root.cgrp`
  - `init_task` belongs to the root cgroup
    - `init_task.cgroups` points to `init_css_set`
- `rstat.c`
  - after a stat is updated, `css_rstat_updated` adds `css->rstat_cpu` to
    `ss->lhead` to mark it dirty
  - before a stat is read, `css_rstat_flush` folds the per-cpu counters into
    the cgroup counter
    - `css_rstat_updated_list`
      - `css_process_update_tree` builds an update tree from `ss->lhead` list
      - it flattens the update tree to an ordered list
    - `cgroup_base_stat_flush` folds the stat
- `namespace.c`
  - `copy_cgroup_ns` returns the old cgroup ns or allocs a new one depending
    on `CLONE_NEWCGROUP`
    - the new cgroup ns treats the task's cgroup as the root cgroup
- `freezer.c`
  - when userspace writes to `cgroup.freeze`, `cgroup_freeze` freezes all
    processes in the cgroup hierarchy

## Userspace

- <https://docs.kernel.org/admin-guide/cgroup-v2.html>
- Introduction
  - Terminology
    - singular "cgroup" refers to the whole feature
    - plural "cgroups" refers to individual control groups
  - What is cgroup?
    - cgroups form a tree
    - every process belongs to one and only one cgroup
    - controllers may be selectively enabled/disabled on a cgroup
- Basic Operations
  - Mounting
    - `mount -t cgroup2 none /sys/fs/cgroup`
  - Organizing Processes and Threads
    - initially only root cgroup exists and all processes belong to the root
      cgroup
    - to create a child subgroup, `mkdir $CHILD`
    - to look up between cgroup and process,
      - `cat /proc/$PID/cgroup` shows the cgroup
      - `cat /sys/fs/cgroup/$CHILD/cgroup.procs` shows all processes
  - [Un]populated Notification
    - `cgroup.events`
  - Controlling Controllers
    - `cgroup.controllers` lists available controllers
    - `cgroup.subtree_control` configs/lists enabled controllers
    - when controller `foo` is enabled for `$PARENT`, `$PARENT/$CHILD/foo.*`
      are created
    - top-down constraint
      - resources are distributed top-down
      - a cgroup can further distribute a resources only when its parent has
        distributed the resource to it
    - No Internal Process Constraint
      - either `cgroup.procs` or `cgroup.subtree_control` must be empty
      - iow, processes must belong to leaf cgroups
      - unless root cgroup, because kthreads belong to the root cgroup
      - this does not apply to "threaded" cgroups, where `cgroup.type` is
        `threaded`
  - Delegation
  - Guidelines
    - avoid migrating processes between cgroups, which is expensive
    - avoid name collision with core `cgroup.*` and controller `foo.*`
- Resource Distribution Models
  - Weights
    - range `[1, 10000]`, default 100
    - e.g., `cpu.weight`
  - Limits
    - range `[0, max]`, default max
    - e.g., `io.max`
  - Protections
    - range `[0, max]`, default 0
    - e.g., `memory.low`
  - Allocations
    - range `[0, max]`, default 0
- Interface Files
  - Format
  - Conventions
  - Core Interface Files
    - `cgroup.type`
    - `cgroup.procs`
    - `cgroup.threads`
    - `cgroup.controllers`
    - `cgroup.subtree_control`
    - `cgroup.events`
    - `cgroup.max.descendants`
    - `cgroup.max.depth`
    - `cgroup.stat`
    - `cgroup.stat.local`
    - `cgroup.freeze`
    - `cgroup.kill`
    - `cgroup.pressure`
- Controllers
  - CPU
    - it regulates distribution of cpu cycles
      - well, it distributes scheduler times actually
      - there is uclamp to hint min/max cpu freq
    - `cpu.weight`
    - `cpu.pressure`
    - `cpu.uclamp.min`
    - `cpu.uclamp.max`
  - Memory
    - it regulates distribution of memory
      - while not all, all major memory usages are tracked
    - `memory.high`
    - `memory.stat`
    - `memory.pressure`
  - IO
    - it regulates distribution of io resources
  - PID
    - it regulates max number of tasks
  - Cpuset
    - it regulates cpu and memory node placement
  - Device controller
    - it regulates access to dev nodes
  - RDMA
  - DMEM
  - HugeTLB
  - Misc
  - Others
  - Non-normative information
- Namespace
  - Basics
  - The Root and Views
  - Migration and setns(2)
  - Interaction with Other Namespaces
- Information on Kernel Programming
  - Filesystem Support for Writeback
- Deprecated v1 Core Features
- Issues with v1 and Rationales for v2
  - Multiple Hierarchies
  - Thread Granularity
  - Competition Between Inner Nodes and Threads
  - Other Interface Issues
  - Controller Issues and Remedies
