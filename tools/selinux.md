SELinux
=======

## Setup

- enable `pam_selinux.so` to set up
  - execution security context for the following `execve`
  - file security context for the controlling terminal
  - security context for new kernel keyring
- `make bare` cleans up
- `make conf` generates `policy/booleans.conf` and `policy/modules.conf`
- `make install`

## Reference Policy

- <https://github.com/SELinuxProject/refpolicy.git>
- `m4` processes the source files
  - `define(foo, hello world)` expands `foo` to `hello world`
  - `policy/users` to `system.users`
  - `.spt` is a support file
  - `.te` is a type enforcement file
  - `.fc` is a file context file
  - `.if` is an interface file
- `.spt` defines low-level stuff
  - `define(interface, ...)` expands `interface`, which is used in `.if`
  - `define(gen_require, ...)` expands `gen_require`, which puts the arg in a
    `require { ... }` block
     - this tells `checkpolicy` to import stuff from another file
  - `define(gen_user, ...)` expands `gen_user` to `user ...` to declare a user
  - `define(gen_context, ...)` expands `gen_context` to the arg
- `.if` defines interfaces
  - `interface(init_daemon_domain, ...` expands `init_daemon_domain`
    - `gen_require` imports `initrc_t`, `system_r`, and `daemon`
    - `typeattribute` adds the specified type to attr `daemon`
    - `domain_type` adds the specified type to attr `domain` and more
    - `domain_entry_file`
    - `role system_r types $1;`
- `.te` defines types, attributes, and rules
  - `attribute domain;` declares new attr `domain`
    - an attr can be assigned to types
    - when a rule refers to an attr, it applies to all types that have the
      attr
      - iow, an attr is a group of types
    - in refpolicy
      - each type having attr `domain` represents a process
      - each type having attr `file_type` represents a file on fs
  - `type httpd_t, domain;` delares new type `httpd_t` and associates it with
    attribute `domain`
    - common process types: `sshd_t`, `auditd_t`, `init_t`
    - common file types: `httpd_sys_content_t`, `etc_t`, `device_t`, `proc_t`
  - `role system_u;` declares new user `system_u`
    - a user is an identity
    - a user is associated with one or more roles
  - `role system_r;` declares new role `system_r`
    - a role is assocaited with one or more types
- `.fc` defines file contexts
  - `/var/log/httpd(/.*)?    gen_context(system_u:object_r:httpd_log_t,s0)`
    - this tells `restorecon` to apply the specified `user:role:type:level` to
      the specified files
    - `ls -Z` shows the file context
