Audit
=====

## audit spam

- PAM uses libaudit to send messages to audit
  - `NETLINK_AUDIT`
- `audit_receive` calls `audit_receive_msg`
  - the message type is in the user ranges
    - `AUDIT_USER`
    - `AUDIT_FIRST_USER_MSG..AUDIT_LAST_USER_MSG`
    - `AUDIT_FIRST_USER_MSG2..AUDIT_LAST_USER_MSG2`
  - `audit_log_user_recv_msg` adds pid, uid, auid, and ses
  - kauditd is woken up
- kauditd calls `kauditd_send_queue` on the main queue
  - the message is sent via `kauditd_send_multicast_skb` to systemd-journald
  - if userspace auditd is not running, the message is also sent via
    `kauditd_hold_skb` where `kauditd_printk_skb` prints it to prink
    - auditd sends a `AUDIT_SET` message that is handled by `auditd_set` to
      register auditd
