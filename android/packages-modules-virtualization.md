# AVF

## CLI: `/apex/com.android.virt/bin/vm`

- `vm info`
  - it supports protected VMs and unprotected VMs
    - there is pkvm managing VMs
    - android itself runs inside a VM
    - android can poke unprotected VMs but not protected VMs
  - `microdroid` is a VM image based on android
- `vm run-microdroid` starts a VM using `microdroid` image
  - `--protected` starts a protected instead of an unprotected VM
  - `--mem 1024` allocs 1GB ram to the VM
  - `--debug none` uses production ramdisk and disables adbd inside VM
  - `--ephemeral` disables secretkeeper integration
  - the work dir is `/data/local/tmp/microdroid/<random>`
    - `storage.img` is a disk image for storage
