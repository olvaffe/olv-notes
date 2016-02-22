suppose ticks happen at t1, t2, and t3 consecutively.

* when t1 happens, we are stuck in another interrupt
* t1 is acked and served just before t2, and not finished until after t2
* at t2, SRCPND and INTPNT is set, yet irq is disabled in cpu core
* when t1 finishes, t2 happens immediately, which finishes before t3
* everything back to normal
* when in oneshot mode, just before t1 finishes, it sees t2 expired and catch up
* if it finishes before t3, t3 is scheduled as next tick
* if not, it explodes because time warps
* does periodic mode explode?
* if t2 finishes after t3, it goes well because t2 has been acked and no time warping
* what if t1 finishes after t3, i.e., it lasts longer than one tick?
* we lost one tick and that's almost all

* clock
  pclk -> timers -> pwm-scaler0 -> pwm-tdiv[0,1]   -> pwm-tin[0,1]
                \                  pwm-tclk0      -/
                 -> pwm-scaler1 -> pwm-tdiv[2,3,4] -> pwm-tin[2,3,4]
                                   pwm-tclk1      -/

  no parent:
  pwm-tclk0
  pwm-tclk1
  
  pwm-tin might have parent pwm-tclk or pwm-tdiv
  
  scaler's set_rate: find a suitable dividor such that parent_rate / divisor == requested
           get_rate: parent_rate / divisor
  dividier is similar

* clocksource always has an error (e.g., read() is called, reg is read, and read() returns happen at different time)
* the error must be acceptable
* errors must not accumulate over time (e.g. delays on every latch reprogramming)
* TCNT = N -> period = N+1, WTCNT = N -> period = N.  Different behavior!

possible combination:

struct clocksource pwm_clocksource = {
};
struct clock_event_deviec pwm_clockevent = {
};
struct clocksource combined_clocksource = {
};
struct clock_event_deviec combined_clockevent = {
};

PWM4:
* current impl.
* moderate clocksource and clockevent
* no special function
* timekeeping needs base

PWMn+PWM4: 
* PWM4 as clocksource, max. latch, but high freq. for timekeeping.  freq chosen for moderate (< HZ) interrupts.
* PWMn as clockevent
* allow for high-res timer
* when 32bits, timekeeping does not need base and nohz works

WDT+PWM4:
* when 32bits, just like PWMn+PWM4
* otherwise, ...
* combined for clocksource and clockevent
* high-res and no_hz

IMPL:
* pwm_clock_register registers a clocksource with moderate interrupts on 16bits and no interrupts on 32bits.


static void pwm_clock_mangle(int dual)
{
    decide_read_func; // read with or without base
    decide_resume_func; // dual not need resume
    modify_cs; // continuous, blah
}

static void pwm_clock_register(int id, int internal_cs, int dual)
{
  if (internal_cs) {
    highest_freq;
    highest_latch;
    no_interrutp;
    return;
  }

  if (dual) {
    freq_has_been_decided_by_clockevent;
    latch_has_been_decided_by_clockevent;
    irq_handler_has_been_setup;
  } else {
    decide_freq_for_moderate_interrupt;
    highest_latch;
    setup_interrupt;
  }

  pwm_clock_mangle(mixed);
  register_cs;
}

* PWM4 -> PWMn+PWM4 (non-dual) -> 32bit -> WDT+PWM4 (non-dual) -> WDT+PWM4 (combined)

* (compromise) find a smallest prescaler * divider, such that HZ is possible
  - smallest -> highest possible timesource resolution
* (hrt) two timers.  one runs pclk/3/2 as clocksource.  the other runs pclk/3/16 as clockevents
  - the first is a clocksource, around 130 interrupts per second, ~0 for s3c6xxx
  - in oneshot mode, 16~HZ (or more) interrupts per-second, depending on load
  - in s3c6xxx, pclk/1/2 should be good for both
  - and it is nohz
  - formula: prescaler and clockevent divider such that interrupts of clock < 2 * HZ / 3

    (pclk / N) / TICK_MAX <= HZ * 2 / 3 -> N = 0 or 6
    dividier = 2 for clock, prescaler = N / 2
    if (prescaler) {
     divider for timer = 16
    } else {
      prescaler = 1
      divider for timer = 2
    }

* (nohz) pwm4 + watchdog
  - pwm4 runs pclk/2, without interrupt
  - watchdog should allow long delay

pwmx + pwm4 nohz not easy because..
* though pwmx allows long enough delay, pwmx and pwm4 might share prescaler
* reprogram of pwmx takes longer than watchdog, time skew
