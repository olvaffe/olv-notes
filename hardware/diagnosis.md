HW Diagnosis
============

## CPU: `turbostat`

- x86-only
- Core i5-5200U
  - <https://www.intel.com/content/www/us/en/products/sku/85212/intel-core-i55200u-processor-3m-cache-up-to-2-70-ghz/specifications.html>
  - Core: Broadwell, 14nm, x2, 2.2GHz boosted to 2.7GHz
  - TDP: 15W
  - RAPL metrics
    - `stress -c 4`: pkg 12.5W (core 11W@2.5GHz), dram 1W
    - `ninja`: pkg 14.5W (core 12.5W@2.5GHz), dram 1.5W
    - shadertoy: pkg 15W (core 1W@1.0GHz, uncore 5W), dram 1W
    - `ninja` plus shadertoy
      - first 20s: pkg 25W (core 21W@2.2GHz, uncore 1.5W), dram 1.5W
      -      then: pkg 15W (core 10W@1.2GHz, uncore 1.5W), dram 1.5W
- Core i7-1185G7
  - <https://www.intel.com/content/www/us/en/products/sku/208664/intel-core-i71185g7-processor-12m-cache-up-to-4-80-ghz-with-ipu/specifications.html>
  - Core: Willow Cove, 10nm, x4, 3.0GHz boosted to 4.8GHz
  - TDP: 28W
  - RAPL metrics
    - `ninja`
      - 0..5s: pkg 35W (core 32W@3.3GHz)
      -  then: pkg 15W (core 12W@2.2GHz)
    - shadertoy: pkg 22W (core 2W@0.8GHz, uncore 17W@1.35GHz)
    - `ninja` plus shadertoy
      -  0..5s: pkg 36W (core 15W@2.5GHz, uncore 17W@1.35GHz)
      - 6..10s: pkg 26W (core 9W@2.2GHz, uncore 14W@1.35GHz)
      -   then: pkg 15W (core 5W@1.3GHz, uncore 6W@1.35GHz)
- Core Ultra 5 135U
  - <https://www.intel.com/content/www/us/en/products/sku/237328/intel-core-ultra-5-processor-135u-12m-cache-up-to-4-40-ghz/specifications.html>
  - Core: 4nm, x2 (1.6GHz boosted to 4.4GHz), x8 (1.1GHz boosted to 3.6GHz), x2 (0.7GHz boosted to 2.1GHz)
  - TDP: 15W
  - RAPL metrics
    - `stress-ng -c 14`
      - 0..60s: pkg 30W (core 26W)
      -   then: pkg 15W (core 12W)
    - shadertoy: pkg 15W (core 0.5W, uncore 9.5W)
    - `stress-ng -c 14` plus shadertoy
      - 0..30s: pkg 34W (core 24W, uncore 6W)
      -   then: pkg 15W (core 8.5W, uncore 3W)

## Memory

- when benchmarking with `memset`,
  - the first iteration faults pages in and should be excluded
  - we should use the dst (`dst[0] *= 2`) otherwise compiler might optimize
    it away
  - we should report the highest bw rather than the average bw
    - in this case, the first iteration is automatically excluded
- when benchmarking with `memcpy`,
  - similar to `memset`
  - additionally, we should init src to non-zero value
    - without init, or when init to zero, kernel might page in the zero page
- `/lib64/ld-linux-x86-64.so.2 --list-tunables`
  - `memset` and `memcpy` skip caches when the size is larger than thresholds
  - `GLIBC_TUNABLES=glibc.cpu.x86_memset_non_temporal_threshold=0xffffffff:glibc.cpu.x86_non_temporal_threshold=0xffffffff`

## Storage: `fio`

- config
  - `filename=/dev/sda`
  - `rw=read`, to avoid corrupt `/dev/sda`
  - `runtime=5`
  - `time_based`
  - `ioengine=libaio`
  - `direct=1`
  - `bs` from `/sys/block/sda/queue/optimal_io_size` and/or experiment
  - `iodepth` from `/sys/block/sda/device/queue_depth` and/or experiment
- nVME: `bs=512K` and `iodepth=16`
  - `READ: bw=3317MiB/s`
- SATA SSD: `bs=4M` and `iodepth=8`
  - `READ: bw=515MiB/s`
- eMMC: `bs=512K` and `iodepth=8`
  - `READ: bw=171MiB/s`
- USB
  - `sudo lspci -vv -s <usb-controller>` and look for `LnkSta`
    - `Speed 5GT/s, Width x1` is pcie 2.0 x1, 0.5GB/s
  - `sudo lsusb -vv -s <adapter>`
    - `Negotiated speed: SuperSpeed (5Gbps)` is usb 3.0
- USB Mass Storage: `bs=512K` and `iodepth=4`
  - `READ: bw=251MiB/s`
- USB UAS over NVMe: `bs=256K` and `iodepth=8`
  - `READ: bw=372MiB/s`
- USB UAS over SATA HDD: `bs=256K` and `iodepth=8`
  - `READ: bw=130MiB/s`
