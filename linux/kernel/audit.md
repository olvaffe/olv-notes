Audit
=====

## audit spam

- audit enablement
  - `audit=` param sets `audit_enabled` at boot time
    - `audit_enabled` defaults to `AUDIT_OFF`
    - `audit=0` also sets `audit_initialized` to `AUDIT_DISABLED`, to disable
      the subsystem entirely
  - userspace sends `AUDIT_SET` msg to set `audit_enabled` at runtime, if the
    subsystem is not disabled
    - auditd always enables audit
    - systemd-journald enables audit when there is a socket and `Audit=yes` in
      `journald.conf`
      - `systemd-journald-audit.socket` is always enabled before 253
      - `Audit=yes` is the default
- PAM uses libaudit to send messages to audit
  - `NETLINK_AUDIT`
- `audit_receive` calls `audit_receive_msg`
  - the message type is in the user ranges
    - `AUDIT_USER`
    - `AUDIT_FIRST_USER_MSG..AUDIT_LAST_USER_MSG`
    - `AUDIT_FIRST_USER_MSG2..AUDIT_LAST_USER_MSG2`
  - `audit_log_user_recv_msg` adds pid, uid, auid, and ses
  - kauditd is woken up
- `kauditd_thread` calls `kauditd_send_queue` on the main queue
  - `kauditd_send_multicast_skb` hook sends the message to multicast clients,
    such as systemd-journald
  - if userspace auditd is running, `netlink_unicast` sends the message to
    auditd
    - auditd sends a `AUDIT_SET` message that is handled by `auditd_set` to
      register auditd
  - if userspace auditd is not running, `kauditd_hold_skb` hook calls
    `kauditd_printk_skb` to print it to printk
  - IOW, as long as audit is enabled and auditd is not running, there will be
    spam
