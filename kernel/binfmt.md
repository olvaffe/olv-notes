Linux binfmt
============

## `binfmt_misc`

- it allows userspace to register an interpreter for any binary
  - `mount -t binfmt_misc none /proc/sys/fs/binfmt_misc`
  - `echo <rule> > /proc/sys/fs/binfmt_misc/register`
- since 4.8, `<rule>` can a `fix binary (F)` flag
  - it tells the kernel to open the interpreter immediately when registered
  - this way, when a binary is invoked, the opened interpreter is used
  - without the flag, the interpreter is opened on binary invocation.  It
    can use a different interpreter if chroot
- on debian, `qemu-user-static` registers the rules with the `fix binary` flag
