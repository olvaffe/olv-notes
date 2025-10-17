Linux Tools
===========

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
