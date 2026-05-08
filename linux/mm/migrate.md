# Kernel Memory Migration

## Overview

- `migrate_pages` migrates a list of folios
  - it updates pgtables and is transparent to the userspace
  - `from` is the list of folios
    - they must have been isolated (removed) from the lru lists
  - `get_new_folio` is a callback to alloc a new target folio
    - it is typically `alloc_migration_target` with `migration_target_control`
  - `reason` is `migrate_reason`
    - `MR_COMPACTION` for memory compaction
    - `MR_MEMORY_FAILURE` for faulty memory offline
    - `MR_MEMORY_HOTPLUG` for memory hot removal
    - `MR_SYSCALL` for `migrate_pages`/`move_pages` syscalls
    - `MR_MEMPOLICY_MBIND` for `mbind` syscall
    - `MR_NUMA_MISPLACED` for numa balancing
    - `MR_CONTIG_RANGE` for `alloc_contig_range` from cma
    - `MR_LONGTERM_PIN` for `pin_user_pages(FOLL_LONGTERM)`
    - `MR_DEMOTION` for reclaim to demote to slower memory tier
    - `MR_DAMON` for policy-based memory tier promote/demote
  - at the end, `migrate_folio_done` drops the old folio
