* 2.6.16 hrtimers
* 2.6.21 NO_HZ
* 

OLD: everything relies on per-arch hardware timer ISR:
* invokes profile_tick
* invokes do_timer
* invokes update_process_times, and timers are fired (including hrtimers)

Clock Event Device:
* per-arch clock event device(s):
* event_handler from generic clock event is called on interrupt
* which does the same thing as the old mechanism

nohz:
* periodic interrupt could be disabled when idle
* catch up jiffies when the system becomes busy again
* interrupts should also start again when the nearest timer expires
* it is easier to switch to oneshot mode and program to fire when the nearest timer expires
* and with a event_handler which programs itself to emulate periodic interrupt

hires:
* periodic interrupt no longer (accurate) enough
* oneshot interrupt for the next_event is needed
* a hrtimer which fires every jiffy is installed, which does what the old mechanism does

summary:
* both nohz and hires requires oneshot mode
* hires is more powerful than nohz

Hrtimers and Beyond
http://tglx.de/projects/hrtimers/ols2006-hrtimers.pdf

GOAL: high-resolution timers, dynamic ticks
OLD: (HW | Clock source | TOD (time of day)) <-> timekeeping
OLD: (HW | Clock event source | ISR) -> tick

Abstraction:

* Clock Source Management
  - abstract read-only clock sources
  - each clock souce expresses the time as a monotonically increasing value
  - helper functions exist to convert clock source's value into nanosecond
  - possible to list or choose from available sources
* Clock Synchronization
  - reference time source (if available) is used to correct the monotonically increasing value
  - the value is read-only and thus the correction is a offset to it
* Time-of-day Representation
* Clock Event Management
  - currently, events defined to be periodic, with fixed period defined at compile time
  - which event sources to use is hardwired arch-specific
  - should abstract clock event sources
* Removing Tick Dependencies
  - current timers assumed a fixed-period periodic event source

CTW (cascade timer wheel)
* the unit is jiffies
* possible timer values in first category: 1, 2, 3, ..., 256 jiffies
* second category: 257, 513, 769, ..., 16384 jiffies
* on tick, timers might be moved, with IRQ disabled!
* moving timers from second category to the first might be expensive
* good performance, but has a bad worst case performance
* suit the need of networking and filesystem

hrtimers (2.6.16)
* those usually not expired (e.g., timer handle io timeout) could still use CTW
* the unit is nanosecond
* timers are kept in a time-sorted, per-cpu, rb-tree
* CLOCK_MONOTONIC and CLOCK_REALTIME
* not really high resolution due to softirq

GTOD:
* Clock Source Management
* Clock Synchronization
* Time-of-day Representation

Clock Event Management:
* foundation for high resolution timers and dynamic tick
* clock event source describes its properties, per-cpu or global
* framework decides which function(s) are provided by which clock event source
* periodic or oneshot modes.  And emulated periodic mode
* event is distributed to periodic tick, process acct., profiling, and next event interrupt (e.g. hi-res timers and dyn tick)

* Documentation/rtc.txt
* Documentation/timers/*

Hardware:
* CMOS Clock.  Could be programmed to alarm.  IRQ 8.
* TSC: Time Stamp Counter, starting Pentium.  Same freq as the CPU.
* PIT: Programmable Interval Timer.  Issue periodic irq on IRQ 0.  HZ ticks per second.
* CPU local timer.  Part of local APIC.  Interupt only a local CPU.
* HPET.  Registers are memory mapped.  Expect to replace PIT.  Interrupt ?.
* ACPI PMT.
