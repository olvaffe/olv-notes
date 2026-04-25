# Linux NUMA

## Initialization

- global `nodemask_t node_states[NR_NODE_STATES]` is statically-initialized
  - `N_POSSIBLE` is `NODE_MASK_ALL`
  - `N_ONLINE` has node 0
  - `N_NORMAL_MEMORY` has node 0
  - `N_MEMORY` has node 0
  - `N_CPU` has node 0
- global `struct pglist_data *node_data[MAX_NUMNODES]` is zero-initialized
  - there will be one `pglist_data` for each `N_ONLINE` node
- x86 `x86_numa_init`  calls its `numa_init(dummy_numa_init)`
- generic `arch_numa_init` calls generic `numa_init(dummy_numa_init)`
- `numa_init(dummy_numa_init)`
  - `numa_memblks_init`
    - it clears
      - `numa_nodes_parsed`
      - `node_possible_map`, aka `node_states[N_POSSIBLE]`
      - `node_online_map`, aka `node_states[N_ONLINE]`
      - `numa_meminfo`
    - `dummy_numa_init`
      - `numa_add_memblk` adds dram to `numa_meminfo`
      - it sets node 0 in `numa_nodes_parsed`
      - x86 also sets node 0 in `numa_phys_nodes_parsed`
    - `numa_register_meminfo`
      - `node_possible_map` is initialized from `numa_nodes_parsed` and
        `numa_meminfo`
      - `memblock_set_node` assigns memblocks to node
  - `numa_register_nodes`
    - `alloc_node_data` allocs pgdata and points `node_data[nid]` to it
    - `node_set_online` adds the node to `node_states[N_ONLINE]`
- without `CONFIG_NUMA`,
  - x86 `x86_numa_init` is nop
  - generic `arch_numa_init` is nop
  - `NODE_DATA` returns `contig_page_data` instead of `node_data`

## `struct pglist_data` aka `pg_data_t`

- each online node has a `pg_data_t`
  - `node_zones` are zones in this node
  - `__lruvec` is lruvec in this node
  - `per_cpu_nodestats` and `vm_stat`
    - there are `NR_VM_NODE_STAT_ITEMS` items
    - stats are updated in `per_cpu_nodestats` and folded into `vm_stat`
- `mm_core_init_early -> free_area_init -> free_area_init_node` inits a pgdat
