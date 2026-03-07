Stop Machine
============

## What

- it does not shutdown the machine
- it schedules the highest priority task to call the specified function
  - `stop_one_cpu` does this to one cpu
  - `stop_machine` does this to all cpus
