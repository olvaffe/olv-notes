Kernel proc
===========

## `/proc/meminfo`

- `meminfo_proc_show`
- `MemTotal`
  - this shows `_totalram_pages`
  - it is increased when free pages are handed over from memblock to buddy
    allocator after early boot
  - it can also be increased on memory hotplug
- `MemFree`
  - this shows `NR_FREE_PAGES`
  - it is modified by `__mod_zone_freepage_state` called from the buddy
    allocator
- `MemAvailable`
  - this shows `si_mem_available`, which is an estimation of memory available
    to userspace before the kernel needs to swap
  - e.g., free pages plus some pagecache plus some reclaimable
- `Buffers`
  - this shows `nr_blockdev_pages`, which is the amount of pagecache used for
    block devices (rather than files)
  - that is, pagecache used for direct blkdev io 
    - e.g., when an fs accesses superblocks such as in `find / -name whatever`
- `Cached`
  - this shows the amount of pagecache used for files
  - `NR_FILE_PAGES` minus `Buffers` minus `SwapCached`
  - `NR_FILE_PAGES` is commonly modified by `__lruvec_stat_mod_folio` called
    from pagecache (lru) or shmem
- `SwapCached`
  - this shows `NR_SWAPCACHE`
  - swap cache consists of pages that have been written to the swap device but
    also exist in page cache
    - this can happen with pages shared by multiple processes
    - when a page is swapped out in one address space, it can still be mapped
      in another address space and must stay in page cache for a little longer
- `Active`
  - this is the sum of `LRU_ACTIVE_ANON` and `LRU_ACTIVE_FILE`
- `Inactive`
  - this is the sum of `LRU_INACTIVE_ANON` and `LRU_INACTIVE_FILE`
- `Active(anon)` / `Inactive(anon)` / `Active(file)` / `Inactive(file)` / `Unevictable`
  - they show `LRU_INACTIVE_ANON`, `LRU_ACTIVE_ANON`, `LRU_INACTIVE_FILE`,
    `LRU_ACTIVE_FILE`, and `LRU_UNEVICTABLE`
  - the pagecache (`struct lruvec`) maintains 5 lists for pages
  - when pages are moved from one list to another, `update_lru_size` is called
    to update the stats
  - active pages are recently used and are less likely to be reclaimed
- `Mlocked`
  - this shows `NR_MLOCK`, pages that are `mlock`ed
- `SwapTotal`
  - this shows `total_swap_pages`
  - it is increased/decreased when swaps are added/removed
- `SwapFree`
  - this shows `nr_swap_pages`, for swap space available
- `Zswap` / `Zswapped`
  - they how `zswap_pool_total_size` and `zswap_stored_pages`
  - zswap receives pages to be swapped out, compresses and stores them in a
    in-memory pool, and swaps compressed pages out when the pool gets full
- `Dirty`
  - this shows `NR_FILE_DIRTY`
  - it is updated when `filemap_dirty_folio` marks pages dirty
- `Writeback`
  - this shows `NR_WRITEBACK`, which are pages being written back
  - that is, pages between `__folio_start_writeback` and
    `__folio_end_writeback`
- `AnonPages`
  - this shows `NR_ANON_MAPPED`
  - it is updated by `page_add_anon_rmap` and `page_remove_rmap`
- `Mapped`
  - this shows `NR_FILE_MAPPED`
  - it is updated by `page_add_file_rmap` and `page_remove_rmap`
- `Shmem`
  - this shows `NR_SHMEM`
  - it is updated by `shmem_add_to_page_cache` and
    `shmem_delete_from_page_cache`
- `KReclaimable`
  - this shows `NR_SLAB_RECLAIMABLE_B` plus `NR_KERNEL_MISC_RECLAIMABLE`
  - `NR_KERNEL_MISC_RECLAIMABLE` has no user anymore
- `Slab`
  - this shows `NR_SLAB_RECLAIMABLE_B` plus `NR_SLAB_UNRECLAIMABLE_B`
- `SReclaimable`
  - this shows `NR_SLAB_RECLAIMABLE_B`
- `SUnreclaim`
  - this shows `NR_SLAB_UNRECLAIMABLE_B`
- `KernelStack`
  - this shows `NR_KERNEL_STACK_KB`
  - it is modified by `account_kernel_stack` called from `dup_task_struct`,
    `do_exit`, etc.
- `PageTables`
- `NFS_Unstable`
- `Bounce`
- `WritebackTmp`
- `CommitLimit`
- `Committed_AS`
- `VmallocTotal`
- `VmallocUsed`
- `VmallocChunk`
- `Percpu`
- `HardwareCorrupted`
- `AnonHugePages`
- `ShmemHugePages`
- `ShmemPmdMapped`
- `FileHugePages`
- `FilePmdMapped`
- `Hugepagesize`
- `Hugetlb`
- `DirectMap4k`
- `DirectMap2M`
- `DirectMap1G`

## `/proc/<pid>/status`

- `/proc/<pid>/status`
  - this is human-readable form of `/proc/<pid>/stat`
  - `VmSize` is `mm->total_vm`
    - `VmData` is `mm->data_vm`
      - sum of vmas that are `VM_WRITE & ~VM_SHARED & ~VM_STACK`
    - `VmStk` is `mm->stack_vm`, size of `VM_STACK` vma
  - `VmRSS` is sum of non-swap `mm->rss_stat` counters
    - sum of `RssAnon`, `RssFile`, and `RssShmem`
  - `VmExe` and `VmLib` sum up to `mm->exec_vm`
    - sum of vmas that are `VM_EXEC & ~VM_WRITE & ~VM_STACK`
  - `VmPTE` is `mm_pgtables_bytes`
  - `VmSwap` is `MM_SWAPENTS` of `rss_stat`
- `pgtables_bytes` in `mm_struct`
  - as page tables are allocated, such as in `pud_alloc`, `mm_inc_nr_*`
    increases `pgtables_bytes`
  - as page tables are freed, such as in `free_pud_range`, `mm_dec_nr_*`
    decreases `pgtables_bytes`
- `rss_stat` in `mm_struct`
  - there are these counters
    - `MM_FILEPAGES`, for file-backed pages
    - `MM_ANONPAGES`, for anonymous pages (e.g., mallocs)
    - `MM_SWAPENTS`, for swapped-out pages
    - `MM_SHMEMPAGES`, for shmem-backed pages
  - the counters are updated by
    - `inc_mm_counter`
    - `dec_mm_counter`
    - `add_mm_rss_vec`
    - `add_mm_counter`
  - as pages are mapped into the mm, `inc_mm_counter` increments the counters
    - `do_set_pte`, which is called on the faulting path, calls
      `inc_mm_counter`
      - if the vma is private, `MM_ANONPAGES` is incremented
      - if the page is swap-backed, `MM_SHMEMPAGES` is incremented
        - shmem has no file-backing and can only be swapped out
      - otherwise, the page is file-backed and `MM_FILEPAGES` is incremented
    - `do_swap_page`, which is called to swap a page in, increments
      `MM_ANONPAGES` and decrements `MM_SWAPENTS`
  - as pages are unmapped from the mm, `dec_mm_counter` decrements the
    counters
    - `try_to_unmap_one`, which is called to unmap a page, calls
      `dec_mm_counter`
      - if the page is anonymous, it can only be swapped out.  `MM_ANONPAGES`
      - if the page is shmem-based, `MM_SHMEMPAGES` is decremented
      - otherwise, the page is file-backed and `MM_FILEPAGES` is decremented

## stats

- `malloc(512MB)` without touching it
  - `ps`: VSZ increased by 512MB
  - `free`: no change
  - `smaps_rollup`: no change
  - `/proc/meminfo`: no change
- `memfd(512MB)` without touching it
  - `ps`: no change
  - `free`: no change
  - `smaps_rollup`: no change
  - `/proc/meminfo`: no change
- `vkAllocateMemory(512MB) on i915` without touching it
  - `ps`: no change
  - `free`: no change
  - `smaps_rollup`: no change
  - `/proc/meminfo`: no change
- `vkAllocateMemory(512MB) on amdgpu` without touching it
  - `ps`: no change
  - `free`: no change
  - `smaps_rollup`: no change
  - `/proc/meminfo`: no change
- `malloc(512MB)` followed by memset
  - `ps`: VSZ and RSS increased by 512MB
  - `free`
    - free and available decreased by 512MB
    - used increased by 512MB
  - `smaps_rollup`
    - these increased by 512MB
      - `Rss`
      - `Pss`
      - `Pss_Anon`
      - `Private_Dirty`
      - `References`
      - `Anonymous`
    - `AnonHugePages` increased by 330MB
    - in `smaps`, there is a mapping with inode 0
  - `/proc/meminfo`
    - these decreased by 512MB
      - MemFree
      - MemAvailable
    - these increased by 512MB
      - Inactive
      - Inactive(anon)
      - AnonPages
      - Committed_AS
    - AnonHugePages increased by 330MB
- `memfd(512MB)` followed by memset
  - `ps`: VSZ (after mmap) and RSS increased by 512MB
  - `free`
    - free and available decreased by 512MB
    - shared and buff/cache increased by 512MB
  - `smaps_rollup`
    - these increased by 512MB
      - `Rss`
      - `Pss`
      - `Pss_Shmem`
      - `Private_Dirty`
      - `References`
    - in `smaps`,
      - there is `/memfd:test (deleted)` with Size 512MB after mmap
      - after memset, these increased: `Rss` `Pss` `Private_Dirty` `References`
  - `/proc/meminfo`
    - these decreased by 512MB
      - MemFree
      - MemAvailable
    - these increased by 512MB
      - Cached
      - Inactive
      - Inactive(anon)
      - Mapped
      - Shmem
      - Committed_AS
    - AnonHugePages did not change
- `vkAllocateMemory(512MB) on i915` followed by memset
  - `ps`
    - VSZ (after vkMapMemory) increased by 512MB
    - RSS did not change
  - `free`
    - free and available decreased by 512MB
    - shared and buff/cache increased by 512MB
  - `smaps_rollup`: no change
    - in `smaps`, there is `anon_inode:i915.gem` with Size 512MB after
      vkMapMemory; pages are not accounted for
  - `/proc/meminfo`
    - these decreased by 512MB
      - MemFree
      - MemAvailable
    - these increased by 512MB
      - Cached
      - Unevictable
      - Shmem
      - Committed_AS
    - AnonHugePages did not change
  - `/sys/kernel/debug/dri/0/i915_gem_objects`
    - total (on first line) increased by 512MB
    - the client increased by 512MB
- `vkAllocateMemory(512MB) on amdgpu` followed by memset
  - `ps`
    - VSZ (after vkMapMemory) increased by 512MB
    - RSS did not change
  - `free`: no change
  - `smaps_rollup`: no change
    - in `smaps`, there is `/dev/dri/renderD128` with Size 512MB after
      vkMapMemory; pages are not accounted for
  - `/proc/meminfo`: no change
  - `/sys/class/drm/card0/device`
    - `mem_info_gtt_used` increased by 512MB
    - `mem_info_vram_used`: no change
  - `/sys/kernel/debug/dri/0/`
    - `amdgpu_gem_info` got a 512MB GTT alloc for the client
    - `amdgpu_gtt_mm`'s gtt available decreased by 512MB and usage increased
      by 512MB
    - `amdgpu_vram_mm`: no change
    - `ttm_page_pool`: unused
  - this is because the test picks a GTT heap
    - when it picks a VRAM heap, `mem_info_vram_used` and `amdgpu_vram_mm`
      changed rather than the gtt counterparts
