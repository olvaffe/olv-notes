ARM Processors
==============

## Classic Profile

- ARMv1
  - ARM1, 1985
- ARMv2
  - ARM2, 1986
- ARMv2a
  - ARM3, 1989
- ARMv3
  - ARM6, 1991
  - ARM7
- ARMv4T
  - T stands for Thumb
  - ARM7T, 1994
- ARMv4
  - ARM8, 1996
- ARMv5TE
  - E stands for Enhanced DSP
  - J stands for Jazelle (for Java)
  - ARM9, 1997
  - ARM10
- ARMv6
  - introduced NEON, Thumb-2, and VFPv2
  - ARM11, 2002

# Application profile

- ARMv7-A
  - Cortex-A8, 2005
  - Cortex-A9, 2007
  - Cortex-A5, 2009
  - Cortex-A15, 2010
  - Cortex-A7, 2011
  - Cortex-A12, 2013
  - Cortex-A17, 2014
- ARMv8-A
  - Cortex-A53, 2012
  - Cortex-A57, 2012
  - Cortex-A35, 2015
  - Cortex-A72, 2015
  - Cortex-A34, 2016
  - Cortex-A73, 2016
- ARMv8.1-A
- ARMv8.2-A
  - Cortex-A55, 2017
  - Cortex-A75, 2017
  - Cortex-A76, 2018
  - Cortex-A77, 2019
  - Cortex-A78, 2020
  - Cortex-X1, 2020
- ARMv8.3-A
- ARMv8.4-A
- ARMv8.5-A and ARMv9.0-A
  - Cortex-A510, 2021
  - Cortex-A710, 2021
  - Cortex-A715, 2022
  - Cortex-X2, 2021
  - Cortex-X3, 2022
- ARMv8.6-A and ARMv9.1-A
- ARMv8.7-A and ARMv9.2-A
  - Cortex-A520, 2023
  - Cortex-A720, 2023
  - Cortex-X4, 2023
  - Cortex-A725, 2024
  - Cortex-X925, 2024
- ARMv8.8-A and ARMv9.3-A
  - C1-Nano, 2025 (was A5xx)
  - C1-Pro, 2025 (was A7xx)
  - C1-Premium, 2025 (new)
  - C1-Ultra, 2025 (was X9xx)
- ARMv8.9-A and ARMv9.4-A
- ARMv9.5-A
- ARMv9.6-A
- ARMv9.7-A
- ARMv9.8-A
- ARMv9.9-A
- ARMv10.0-A

## Microcontroller Profile

- ARMv6-M
  - Cortex-M0, 2009
  - Cortex-M0+, 2012
  - Cortex-M1, 2007
- ARMv7-M
  - Cortex-M3, 2004
- ARMv7E-M
  - Cortex-M4, 2010
  - Cortex-M7, 2014
- ARMv8-M
  - Cortex-M23, 2016
  - Cortex-M33, 2016
  - Cortex-M35P, 2018
- ARMv8.1-M
  - Cortex-M55, 2020
  - Cortex-M85, 2022

## Real-time Profile

- ARMv7-R
  - Cortex-R4, 2011
  - Cortex-R5, 2011
  - Cortex-R7, 2011
  - Cortex-R8, 2016
- ARMv8-R
  - Cortex-R52, 2016
  - Cortex-R82, 2020

## Distro Support

- Debian has 3 flavors
  - armel
    - 32-bit
    - soft-float
    - ARMv5TE
    - ARMv6
  - armhf
    - 32-bit
    - hard-float
    - ARMv7
  - arm64
    - 64-bit
    - ARMv8

## Performances

- little
  - 2012, A53, up to 2.3 GHz
  - 2017, A55, up to 2.3 GHz, +18% perf
  - 2021, A510, up to 2.0 GHz, +35% perf
  - 2023, A520, up to 2.0 GHz, +8% perf
- big
  - 2012, A57, up to 2.0 GHz
  - 2015, A72, up to 2.5 GHz, +90% perf
  - 2016, A73, up to 2.8 GHz, +30% perf
  - 2017, A75, up to 3.0 GHz, +20% perf
  - 2018, A76, up to 3.3 GHz, +35% perf
  - 2019, A77, up to 3.3 GHz, +20% perf
  - 2020, A78, up to 3.3 GHz, +7% perf
  - 2021, A710, up to 3.0 GHz, +10% perf
  - 2022, A715, up to 3.0 GHz, +5% perf
  - 2023, A720, up to 3.0 GHz, +15% perf
- x
  - 2019, A77, up to 3.3 GHz
  - 2020, X1, up to 3.3 GHz, +20% perf
  - 2021, X2, up to 3.0 GHz, +30% perf
  - 2022, X3, up to 3.2 GHz, +25% perf
  - 2023, X4, up to 3.4 GHz, +15% perf
- <https://www.cpubenchmark.net/singleThread.html#mobile-thread>
  - X2
    - 3.1GHz, 3111
  - A78
    - 3.0GHz, 2647
    - 2.6GHz, 1764
    - 2.5GHz, 1641
  - A77
    - 3.1GHz, 2687
    - 3.1GHz, 2592
  - A76
    - 2.6GHz, 1823
  - A55
    - 2.1GHz, 336
    - 2.0GHz, 313
    - 1.8GHz, 288
    - 1.4GHz, 230
  - A53, 2012
    - 1.9GHz, 274
    - 1.7GHz, 244
    - 1.6GHz, 218
    - 1.5GHz, 197
    - 1.4GHz, 193
    - 1.3GHz, 183
    - 1.2GHz, 178
