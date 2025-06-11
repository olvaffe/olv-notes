Linux NUMA
==========

## NUMA Nodes

- there is a `struct pglist_data *node_data[MAX_NUMNODES];` array
  - one `pglist_data` for each node
- a `pglist_data` has many fields
  - `node_zones` are zones in this node
  - `__lruvec` is lruvec in this node
  - `vm_stat`
    - there are `NR_VM_NODE_STAT_ITEMS` items

## Zones

- a `zone` has many fields
  - `vm_stat`
    - there are `NR_VM_ZONE_STAT_ITEMS` items

## LRU

- a `lruvec` has many fields
  - `lists`
    - there are `NR_LRU_LISTS` lists
