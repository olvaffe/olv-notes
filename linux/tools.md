Linux Tools
===========

## perf

- `perf record`
  - `perf record -e cycles -c 1000 <cmd>`
    - generate an interrupt and collect a sample for every 1000 cycles
    - every 1000000 cycles is more appropriate for cycles
  - `perf record -e cycles -F 1000 <cmd>`
    - generate 1000 interrupts and colloect 1000 samples every second
- tracepoint events
  - `perf stat -e dma_fence:* <cmd>`
  - `perf stat -e drm:* <cmd>`
  - `perf stat -e kvm:* <cmd>`
  - `perf stat -e power:* <cmd>`
  - `perf stat -e sched:* <cmd>`
  - `perf stat -e syscalls:* <cmd>`
- symbols
  - `/proc/sys/kernel/kptr_restrict`
    - default 0 on arch linux
  - `/proc/sys/kernel/perf_event_paranoid`
    - default 2 on arch linux
  - just use sudo to record and report
  - `perf record` has debuginfod integration
    - when `DEBUGINFOD_URLS` is set, it downloads all symbols and is slow at the
      end of the first run
    - the debuginfod server for arch linux only has debug symbols for the latest
      packages; remember to update/reboot first
- experiment: compare idling and busy looping for 1s
  - `perf stat --timeout 1000 blah`
  - cpu: `i7-7820HQ`
    - `cat /sys/devices/system/cpu/cpu0/cpufreq/base_frequency` is 2900000
    - `cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq` is 3900000
  - HW events
    - branches: 211k vs. 3825M
    - branch-misses: 7k vs. 11k
    - cache-misses: 28k vs. 28k
    - cycles: 1M vs. 3834M
    - instructions: 1M vs. 3827M
  - SW events
    - context-switches: 1 vs. 3
    - cpu-clock: 1.3ms vs. 1000ms
    - page-faults: 63 vs. 45
    - task-clock: 1.3ms vs. 1000ms
- experiment: an idle system and a program in a tight loop that sleeps for
  different times
  - sleep 1us
    - top says 10.5% cpu time
    - perf record says 24.5% cpu cycles
    - perf stat says 20% cpu
  - sleep 10us
    - top says 9% cpu time
    - perf record says 20.5% cpu cycles
    - perf stat says 15% cpu
  - sleep 100us
    - top says 4% cpu time
    - perf record says 11% cpu cycles
    - perf stat says 7% cpu
  - sleep 1ms
    - top says 1.5% cpu time
    - perf record says 7% cpu cycles
    - perf stat says 2% cpu
  - sleep 10ms
    - top says 0.5% cpu time
    - perf record says 3.5% cpu cycles
    - perf stat says 0.3% cpu

## turbostat

- `sudo turbostat -l` lists available counters
  - `usec` shows `gettimeofday` relative to start
  - `Time_Of_Day_Seconds` shows `gettimeofday`
  - `Avg_MHz` is derived from `MSR_APERF`
  - `Busy%` is derived from `MSR_MPERF`
  - `Bzh_MHz` is derived from both `MSR_APERF` and `MSR_MPERF`
  - `TSC_MHz` shows `rdtsc`
  - `IRQ` and `NMI` show `/proc/interrupts` 
  - `SMI` shows `MSR_SMI_COUNT`
  - `CPU%c1`, `CPU%c6`, and `CPU%c7` show `MSR_CORE_Cx_RESIDENCY`
  - `CoreTmp` shows `MSR_IA32_THERM_STATUS`
  - `PkgTmp` shows `MSR_IA32_PACKAGE_THERM_STATUS`
  - `GFX%rc6`, `GFXMHz`, and `GFXAMHz` are derived from the intel gpu driver
    - xe: `/sys/class/drm/card0/device/tile0/*`
    - i915: `/sys/class/drm/card%d/gt/gt0/*`
  - `Pkg%pc2`, `Pkg%pc3`, `Pkg%pc6`, `Pkg%pc7`, `Pkg%pc8`, `Pkg%pc9`, and
    `Pk%pc10` show `MSR_PKG_Cx_RESIDENCY`
  - `CPU%LPI` shows `/sys/devices/system/cpu/cpuidle/low_power_idle_cpu_residency_us`
    - the time the cpu package is in PKG C10
  - `SYS%LPI` shows `/sys/devices/system/cpu/cpuidle/low_power_idle_system_residency_us`
    - the time the cpu package is in PKG C10 and PCH is in low power state
  - `PkgWatt` shows `MSR_PKG_ENERGY_STATUS`
  - `CorWatt` shows `MSR_PP0_ENERGY_STATUS`
  - `GFXWatt` shows `MSR_PP1_ENERGY_STATUS`
  - `RAMWatt` shows `MSR_DRAM_ENERGY_STATUS`
  - `PKG_%` shows `MSR_PKG_PERF_STATUS`
  - `RAM_%` shows `MSR_DRAM_PERF_STATUS`
  - `Totl%C0` shows `MSR_PKG_WEIGHTED_CORE_C0_RES`
  - `Any%C0` shows `MSR_PKG_ANY_CORE_C0_RES`
  - `GFX%C0` shows `MSR_PKG_ANY_GFXE_C0_RES`
  - `CPUGFX%` shows `MSR_PKG_BOTH_CORE_GFXE_C0_RES`
  - `Core` shows `/sys/devices/system/cpu/cpu%d/topology/core_id`
  - `CPU` shows `/sys/devices/system/cpu/cpu%d`
  - `APIC` and `X2APIC` are derived from cpuid
  - `IPC` shows instr count from perf divided by `MSR_APERF`
  - `CoreThr` shows `/sys/devices/system/cpu/cpu%d/thermal_throttle/core_throttle_count`
  - `UncMHz` shows legacy `/sys/devices/system/cpu/intel_uncore_frequency/package_%02d_die_%02d/current_freq_khz`
  - `SysWatt` shows `MSR_PLATFORM_ENERGY_STATUS`
  - `/sys/devices/system/cpu/cpu%d/cpuidle/state%d`
    - `<name>%` is derived from `time`
    - `<name>` is derived from `usage`
    - `<name>+` is derived from `below`
    - `<name>-` is derived from `above`

## cpupower

- `frequency-info` uses `cpufreq` subsys
- `idle-info` uses `cpuidle` subsys
  - on x86, a core can be in `POLL`, `C1_ACPI`, `C2_ACPI`, or `C3_ACPI`
- `powercap-info` uses `powercap` subsys
  - on intel, there are `core` and `uncore`
- `monitor` has various backends
  - `Idle_Stats` uses `cpuidle` subsys to report percents of time each core is
    in each idle states
  - `RAPL` uses `powercap` subsys to report microjoules each core consumes
  - `Mperf` reads `MSR_{A,M}PERF` regs to report cpu busyness
    - `MSR_MPERF` increments at core max freq
    - `MSR_APERF` increments at core cur freq when the core is in `C0`
  - `Nehalem` reads `MSR_{CORE,PKG}_C{3,6}_RESIDENCY` regs to report cpu busyness
    - the regs increment when the core/pkg is in the respective idle states
    - the core ones seem outdated as my cores stay in C7 a lot but never in C6

## tmon

- strictly for thermal subsys
  - no hwmon
- thermal zones (sensors)
  - trip points
    - `C` for `THERMAL_TRIP_CRITICAL`
    - `H` for `THERMAL_TRIP_HOT`
    - `P` for `THERMAL_TRIP_PASSIVE`
    - `A` for `THERMAL_TRIP_ACTIVE`
- cooling devices
