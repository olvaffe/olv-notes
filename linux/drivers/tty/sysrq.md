Kernel SysRq
============

## Initialization

- `sysrq_init_procfs` adds `/proc/sysrq-trigger`
- `sysrq_register_handler` registers `sysrq_handler` to the input subsystem

## ops

- `0` to `9` maps to `sysrq_loglevel_op`
- `b` maps to `sysrq_reboot_op`
  - `emergency_restart`
- `c` maps to `sysrq_crash_op`
  - `panic("sysrq triggered crash\n")`
- `d` maps to `sysrq_showlocks_op`
  - `debug_show_all_locks`
- `e` maps to `sysrq_term_op`
  - `send_sig_all(SIGTERM)`
- `f` maps to `sysrq_moom_op`
  - `schedule_work(&moom_work)`
- `i` maps to `sysrq_kill_op`
  - `send_sig_all(SIGKILL)`
- `j` maps to `sysrq_thaw_op`
  - `emergency_thaw_all`
- `k` maps to `sysrq_SAK_op`
  - `schedule_work(SAK_work)`
- `l` maps to `sysrq_showallcpus_op`
  - `show_regs`
  - `show_stack`
- `m` maps to `sysrq_showmem_op`
  - `show_mem`
- `n` maps to `sysrq_unrt_op`
  - `normalize_rt_tasks`
- `p` maps to `sysrq_showregs_op`
  - `show_regs`
  - `perf_event_print_debug`
- `q` maps to `sysrq_show_timers_op`
  - `sysrq_timer_list_show`
- `r` maps to `sysrq_unraw_op`
  - `vt_reset_unicode(fg_console)`
- `s` maps to `sysrq_sync_op`
  - `emergency_sync`
- `t` maps to `sysrq_showstate_op`
  - `show_state`
  - `show_all_workqueues`
- `u` maps to `sysrq_mountro_op`
  - `emergency_remount`
- `w` maps to `sysrq_showstate_blocked_op`
  - `show_state_filter(TASK_UNINTERRUPTIBLE)`
- `z` maps to `sysrq_ftrace_dump_op`
  - `ftrace_dump(DUMP_ALL)`
