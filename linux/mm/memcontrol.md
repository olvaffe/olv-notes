# Kernel MEMCG

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

## Charging Internals

- callbacks
  - `mem_cgroup_css_alloc` allocs a `mem_cgroup`
  - `mem_cgroup_css_online` onlines a `mem_cgroup`
    - `obj_cgroup_alloc` allocs an `obj_cgroup`
      - subsystems work with light `obj_cgroup` rather than heavy `mem_cgroup`
      - when a cgroup is removed, heavy `mem_cgroup` can be freed and light
        `obj_cgroup` lives on for a bit longer
- `obj_cgroup` vs `mem_cgroup`
  - `objcg->memcg` is a weak pointer to memcg
    - `get_mem_cgroup_from_objcg` promotes it to strong pointer
  - `objcg->nr_charged_bytes` enables sub-page charging
- `page->memcg_data`
  - it used to point to memcg
    - when the memcg is destroyed, it is not freed until the last page loses
      the reference
    - with objcg and weak pointer, the memcg is freed immediately
  - buddy alloc: `alloc_pages`
    - with `__GFP_ACCOUNT`, `__memcg_kmem_charge_page` calls `page_set_objcg`
      to update the pointer with `MEMCG_DATA_KMEM`
    - without `__GFP_ACCOUNT`, the page is not accounted
  - slab alloc: `kmalloc`, `kmem_cache_create`, etc.
    - `KMALLOC_CGROUP` implies `SLAB_ACCOUNT`
    - with `SLAB_ACCOUNT`, `allocate_slab` calls `account_slab` to update the
      pointer with `MEMCG_DATA_OBJEXTS`
      - `slab->obj_exts` and `page->memcg_data` alias with each other
    - without `SLAB_ACCOUNT`, the slab is not accounted
  - user alloc: anon, shmem, page cache, etc.
    - they use buddy alloc, but they do not set `__GFP_ACCOUNT`
    - instead,
      - `alloc_anon_folio` calls `mem_cgroup_charge` explictly
      - `shmem_alloc_and_add_folio` calls `mem_cgroup_charge` explicitly
      - `filemap_add_folio` calls `mem_cgroup_charge` explicitly
    - `mem_cgroup_charge` calls `commit_charge` to update the pointer
- `charge_memcg` charges a folio to a memcg
  - `try_charge_memcg`
    - `consume_stock` consumes the pre-charged quota
      - for speed, we always charge at least `MEMCG_CHARGE_BATCH` (64) pages
    - `page_counter_try_charge` charges to page counter
    - `refill_stock` updates the pre-charged quota
    - if it fails to charge, depending on gfp flags,
      - `try_to_free_mem_cgroup_pages` reclaims
      - `mem_cgroup_oom` oom kills
  - `commit_charge` updates `folio->memcg_data` to point to objcg
- `obj_cgroup_charge` charges bytes to a objcg
  - `consume_obj_stock` consumes the pre-charged quota
    - we always charge in pages
  - `__obj_cgroup_charge` charges whole pages
    - `obj_cgroup_charge_pages` calls `try_charge_memcg`
  - `refill_obj_stock` updates the pre-charged quota

## User Charging

- `mem_cgroup_charge` charges a folio to a user mm
  - `get_mem_cgroup_from_mm` returns the memcg for the mm
    - if mm is known, return `mem_cgroup_from_task(mm->owner)`
      - e.g., when we fault in a folio, `vma->vm_mm` is the mm
      - `mm->owner` is the thread group leader
    - if mm is unknown,
      - if remote charging active, use memcg from `set_active_memcg`
      - if `current->mm`, use it as the mm
        - `current->mm` is the user mm, and is NULL for kthreads
        - fwiw, `current->active_mm` is the active mm used by mmu
      - else, use `root_mem_cgroup`
  - `charge_memcg` charges
- `mem_cgroup_charge_hugetlb` charges an explicit hugetlb folio
  - `get_mem_cgroup_from_current` returns `mem_cgroup_from_task(current)`
  - `charge_memcg` charges

## Swap Charging

- `mem_cgroup_try_charge_swap` is called during swap out
  - since anon map or shmem has charged the folio, `folio_objcg(folio)`
    returns the objcg and `obj_cgroup_memcg` returns the memcg
  - `swap_cgroup_record` saves the memcg to `swap_cgroup_ctrl`
- `mem_cgroup_swapin_charge_folio` is called during swap in
  - `lookup_swap_cgroup_id` and `mem_cgroup_from_private_id` return the
    saved memcg
  - `charge_memcg` charges the saved memcg

## Kernel Charging

- `current_obj_cgroup` returns the objcg to charge
  - if `in_mmi`, return NULL to skip charging
  - if `in_task`,
    - if remote charging, use memcg from `set_active_memcg`
    - if user task, use `current->objcg`
      - it caches `current->cgroups->memcg->nodeinfo[nid]->objcg`
      - if outdated, `current_objcg_update` refreshes
    - if kthread, use `root_mem_cgroup->nodeinfo[nid]->objcg`
  - else, we are in irq
    - if remote charging, use memcg from `set_active_memcg`
    - else, use `root_mem_cgroup->nodeinfo[nid]->objcg`
  - in the case of remote charging, `memcg->nodeinfo[nid]->objcg` goes from
    memcg to objcg
- `memcg_kmem_charge_page` charges whole pages
  - `current_obj_cgroup` returns the objcg
  - `obj_cgroup_charge_pages` charges whole pages
  - `page_set_objcg` overrides `page->memcg_data` to objcg plus `MEMCG_DATA_KMEM`
- `mem_cgroup_sk_charge` charges sk buffer
  - `mem_cgroup_from_sk` returns the memcg
    - when userspace creates/accepts a socket, net core calls
      `mem_cgroup_sk_alloc` to init `sk->sk_memcg` to
      `mem_cgroup_from_task(current)`
  - `try_charge_memcg` charges
- sub-page charging calls `obj_cgroup_charge` and friends
  - e.g., slab
    - `allocate_slab` allocs a slab but does not charge to memcg
      - if `SLAB_ACCOUNT`, `alloc_slab_obj_exts` inits `slab->obj_exts` to
        `slabobj_ext` plus `MEMCG_DATA_OBJEXTS`
        - `slab->obj_exts` overlays `page->memcg_data`
    - `kmalloc` calls `slab_post_alloc_hook` after sub-alloc
      - `memcg_slab_post_alloc_hook` charges to memcg if `__GFP_ACCOUNT` or
        `SLAB_ACCOUNT`

## Stats

- `memcg_stat_item` extends `node_stat_item`
  - the first memcg stat item is after the last node stat item
  - there are holes
    - only those on `memcg_node_stat_items` and `memcg_stat_items` are allowed
    - `BAD_STAT_IDX` returns true for holes
- each tracked page belongs to a memcg
  - `page->memcg_data` points to the memcg
- lruvec is per-cgroup and per-node
  - `mem_cgroup_lruvec` returns the lruvec for the `(memcg, pgdat)` pair
  - when `CONFIG_MEMCG=n`, it is per-node
- why `memcg_stat_item`?
  - for `node_stat_item`, we account for allocated physical pages
    - e.g., `lruvec_stat_mod_folio`
      - `folio_pgdat` returns the pgdat
      - `folio_memcg` returns the memcg
      - `mod_node_page_state` updates the node stat
      - if tracked, `mem_cgroup_lruvec` returns the lruvec
        - `mod_memcg_lruvec_state` updates the lruvec stat
  - for `memcg_stat_item`, we don't require a physical page
    - `mod_memcg_state` updates the memcg stat
  - `MEMCG_SWAP` accounts for `swp_entry_t`
    - it is only read on cgroup v1
  - `MEMCG_SOCK` accounts for `sock`
  - `MEMCG_PERCPU_B` accounts for per-cpu bytes
  - `MEMCG_KMEM` accouts for bytes (and pages)
  - `MEMCG_ZSWAP_B`
  - `MEMCG_ZSWAPPED`
  - `MEMCG_ZSWAP_INCOMP`
