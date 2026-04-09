Kernel MEMCG
============

## What are Charged?

- userspace `malloc` and stack are charged
  - `do_anonymous_page` calls `alloc_anon_folio` which calls
    `mem_cgroup_charge`
  - they are uncharged upon
    - free: `__folio_put` calls `mem_cgroup_uncharge`
    - reclaim: `shrink_folio_list` calls `mem_cgroup_uncharge_folios`
- page cache is charged
  - `filemap_add_folio` calls `mem_cgroup_charge`
- shmem is charged
  - `shmem_alloc_and_add_folio` calls `mem_cgroup_charge`
- vfs cache (inode and dentry) is charged
  - their kmem caches are created with `SLAB_ACCOUNT`
- pgtables are charged
  - pgtables are allocated with `GFP_PGTABLE_USER`, which includes
    `__GFP_ACCOUNT`
- swap is charged
  - `folio_alloc_swap` calls `mem_cgroup_try_charge_swap`
- task structs are charged
  - `task_struct_cachep` is created with `SLAB_ACCOUNT`
- kernel stacks are charged
  - `memcg_charge_kernel_stack` calls `memcg_kmem_charge_page`
- network buffers are charged
  - net core calls `mem_cgroup_sk_charge`
- many more
