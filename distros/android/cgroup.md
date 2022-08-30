Android cgroup
==============

## Overview

- <https://source.android.com/docs/core/perf/cgroups>
  - `/system/core/libprocessgroup/profiles/task_profiles.json`
  - `/system/core/libprocessgroup/profiles/cgroups.json`
- examples
  - `cgroups.json` defines the controllers
    - the `cpuset` controller is at `/dev/cpuset`
    - the `schedtune` controller is at `/dev/stune`
    - the `cpu` controller is at `/dev/cpuctl`
  - an app can be assigned multiple profiles defined in `task_profiles.json`
    - `ProcessCapacityMax` profile adds the app to `cpuset` controller's
      `top-app`
      - this writes the pid of the app to `/dev/cpuset/top-app/tasks`
    - `HighPerformance` profile can add the app to `schedtune` controller's
      `foreground`
      - this writes the pid of the app to `/dev/stune/foreground/tasks`
- manual example
  - `mkdir /dev/cpuctl/top-app`
    - create a group
  - `for i in cpu.rt_{runtime,period}_us; do cp /dev/cpuctl/$i /dev/cpuctl/top-app; done`
    - init some attributes of the group
  - `cp /dev/cpuset/top-app/cgroup.procs /dev/cpuctl/top-app/cgroup.procs`
    - add tasks in `cpuset/top-app` group to the new group
  - `echo 40 > /dev/cpuctl/top-app/cpu.uclamp.min`
    - apply uclamp min to the group
