# Activity Manager

## `dumpsys meminfo`

- `ActivityManagerService::dumpApplicationMemoryUsage`
  - `ServiceManager.addService("meminfo", new MemBinder(this), ...)`
  - `MemBinder::dump` calls `dumpApplicationMemoryUsage`
- `dumpApplicationMemoryUsageHeader` prints
  - `Applications Memory Usage (in Kilobytes):`
  - `Uptime: <ms> Realtime: <ms>`
- `Total RSS by process:`
- `Total RSS by OOM adjustment:`
- `Total RSS by category:`
- `Total PSS by process:`
- `Total PSS by OOM adjustment:`
- `Total PSS by category:`
- `Global Mapped Bitmaps:`
- `Total RAM: <size> (status normal)`
  - `MemInfoReader::readMemInfo` calls `SysMemInfo::ReadMemInfo` to parse
    `/proc/meminfo`
- `DMA-BUF:   <size> (  <size> mapped +    <size> unmapped)`
  - `Debug::getDmabufTotalExportedKb` calls `GetDmabufPerBufferStats` to parse
    android-specific
    - `/sys/fs/bpf/dmabuf/prog_dmabufIter_iter_dmabuf`
      - `system/bpfprogs/dmabufIter.c`
    - `/sys/kernel/dmabuf/buffers`
- `DMA-BUF Heaps:   <size>`
  - `Debug::getDmabufHeapTotalExportedKb` calls
    `ReadDmabufHeapTotalExportedKb`
  - it calls `GetDmabufPerBufferStats` and only includes dma-bufs exported
    from heaps
- `DMA-BUF Heaps pool:   <size>`
  - `Debug::getDmabufHeapPoolsSizeKb` calls `ReadDmabufHeapPoolsSizeKb` to
    parse android-specific `/sys/kernel/dma_heap/total_pools_kb`
- `GPU:   <size> (  <size> dmabuf +     <size> private)`
  - `Debug::getGpuTotalUsageKb` calls `meminfo::ReadGpuTotalUsageKb` to parse
    android-specific `/sys/fs/bpf/map_gpuMem_gpu_mem_total_map`
    - `frameworks/native/services/gpuservice/bpfprogs/gpuMem.c` parses
      `gpu_mem/gpu_mem_total` tracepoint
  - `Debug::getGpuPrivateMemoryKb` calls `memtrack_proc_gl_pss`
- `Kernel CMA:    <size>`
  - `Debug::getKernelCmaUsageKb` calls `meminfo::ReadKernelCmaUsageKb` to
    parse `/sys/kernel/mm/cma`
- `Used RAM: <size> (<size> used pss +   <size> kernel)`
  - for kernel size, it sums various meminfo, gpu, cma, etc.
- `Lost RAM:   <size>`
  - `MemoryUsageStats::getLostRam` derives it from `/proc/meminfo`
- `ZRAM:   <size> physical used for   <size> in swap (<size> total swap)`
  - `MEMINFO_ZRAM_TOTAL`
    - this is non-standard and `SysMemInfo::ReadMemInfo` calls `mem_zram_kb`
      to parse `/sys/block/zram*/mm_stat`
  - `MEMINFO_SWAP_FREE`
  - `MEMINFO_SWAP_TOTAL`
