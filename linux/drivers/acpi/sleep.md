ACPI sleep
==========

## ACPI Power States

- global states and sleep states
  - `G0`: working
    - `S0`
      - all cpu/mem/devices are powered up
  - `G1`: sleeping
    - `S0ix`: modern standby, lower power s0 idle, suspend-to-idle, etc.
      - cpu/mem are powered up, but cpu is in low-power idle
      - devices may be powered up or down
      - should have the same power consumption as `S3`, except allowing
        instant-on similar to a phone/tablet
    - `S1`
      - cpu/mem are powered up, but cpu is in idle
      - devices may be powered up or down
      - intel does not support this state
    - `S2`
      - cpu is powered down
      - mem is powered up
      - devices may be powered up or down
      - intel does not support this state
    - `S3`: standby, sleep, suspend-to-ram, etc.
      - both cpu/devices are powered down
      - mem is self-refreshing
    - `S4`: hibernation, suspend-to-disk, etc.
      - all cpu/mem/devices are powered down
      - the content of the main memory has been saved to nvmem such as disk
  - `G2`: soft off
    - `S5`
      - PSU still supplies power, at least to the power button to return to
        `S0`
      - all cpu/mem/devices are powered down, except for some devices (e.g.,
        LAN) that might remain powered up to wake up the computer
  - `G3`: mechanical off
    - PSU has shut down the power via a mechanical switch
    - the power cord can be removed and the computer can be safely
      disassembled
- processor states
  - `C0`: operating
  - `C1`: halt
    - the cpu executes the halt instruction
  - `C2`: clock-gated
    - the cpu does not execute instructions, but still maintains sw-visible
      states
  - `C3`: sleep
    - the cpu loses some of the sw-visible states
- device states
  - `D0`: fully on
  - `D1`: clock-gated
    - the device is powered up (to preserve the hw context), but the clock is
      gated
  - `D2`
    - the device is partially powered up
    - optional
  - `D3hot`
    - the device is partially powered up, such that it is detectable by the
      bus controller and can return to `D0`
  - `D3cold`
    - the device is powered down and is not detectable by the bus controller
- performance states when a processor/device is in `C0`/`D0`
  - `P0`: max power and freq
  - `P1`: slower than `P0`
  - `P2`: slower than `P1`
  - `Pn`: slower than `P(n-1)`
