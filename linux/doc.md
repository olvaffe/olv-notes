# Kernel Docs

## Repo Overview

- <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git>
  - development happens on `master`
  - `vM.m-rc1` is tagged two weeks after the last release
  - `vM.m-rcN` and eventually `vM.m` follows every Sunday
- <https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git>
  - `linux-M.m.y` branches and `vM.m.y` tags
- <https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git>
  - `stable` points to the latest upstream tag as the baseline
  - `next-YYYYMMDD` is tagged daily
    - it resets `master` to `stable`
    - it merges all tracked branches, listed in `Next/Trees`
    - it tags `master`
- arch
  - arm64: <https://git.kernel.org/pub/scm/linux/kernel/git/arm64/linux.git>
  - arm64/boot/dts: <https://git.kernel.org/pub/scm/linux/kernel/git/soc/soc.git>
  - x86: <https://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git>
- block
  - <https://git.kernel.org/pub/scm/linux/kernel/git/axboe/linux.git>
- crypto
  - <https://git.kernel.org/pub/scm/linux/kernel/git/herbert/crypto-2.6.git>
- drivers
  - accel, dma-buf, gpu: <https://gitlab.freedesktop.org/drm/kernel>
  - acpi, cpufreq, cpuidle, devfreq, idle, opp, pnp, powercap, thermal: <https://git.kernel.org/pub/scm/linux/kernel/git/rafael/linux-pm.git>
  - base: <https://git.kernel.org/pub/scm/linux/kernel/git/gregkh/driver-core.git>
  - block, nvme: <https://git.kernel.org/pub/scm/linux/kernel/git/axboe/linux.git>
  - bluetooth: <https://git.kernel.org/pub/scm/linux/kernel/git/netdev/net.git>
  - char: <https://git.kernel.org/pub/scm/linux/kernel/git/gregkh/char-misc.git>
    - tpm: <https://git.kernel.org/pub/scm/linux/kernel/git/jarkko/linux-tpmdd.git>
  - clk: <https://git.kernel.org/pub/scm/linux/kernel/git/clk/linux.git>
  - clocksource, irqchip, virt: <https://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git>
  - crypto: <https://git.kernel.org/pub/scm/linux/kernel/git/herbert/crypto-2.6.git>
  - dma: <https://git.kernel.org/pub/scm/linux/kernel/git/vkoul/dmaengine.git>
  - firmware:
    - `arm_scmi`: <https://git.kernel.org/pub/scm/linux/kernel/git/soc/soc.git>
    - efi: <https://git.kernel.org/pub/scm/linux/kernel/git/efi/efi.git>
    - google: <https://git.kernel.org/pub/scm/linux/kernel/git/chrome-platform/linux.git>
  - gpio: <https://git.kernel.org/pub/scm/linux/kernel/git/brgl/linux.git>
  - hid: <https://git.kernel.org/pub/scm/linux/kernel/git/hid/hid.git>
  - hwmon, watchdog: <https://git.kernel.org/pub/scm/linux/kernel/git/groeck/linux-staging.git>
  - i2c: <https://git.kernel.org/pub/scm/linux/kernel/git/andi.shyti/linux.git>
  - iio, interconnect, misc, nvmem: <https://git.kernel.org/pub/scm/linux/kernel/git/gregkh/char-misc.git>
  - input: <https://git.kernel.org/pub/scm/linux/kernel/git/dtor/input.git>
  - iommu: <https://git.kernel.org/pub/scm/linux/kernel/git/iommu/linux.git>
  - leds: <https://git.kernel.org/pub/scm/linux/kernel/git/lee/leds.git>
  - mailbox: <https://git.kernel.org/pub/scm/linux/kernel/git/jassibrar/mailbox.git>
  - md: <https://git.kernel.org/pub/scm/linux/kernel/git/device-mapper/linux-dm.git>
  - media: <https://git.kernel.org/pub/scm/linux/kernel/git/mchehab/linux-media.git>
  - memory, reset, soc, tee: <https://git.kernel.org/pub/scm/linux/kernel/git/soc/soc.git>
  - mfd: <https://git.kernel.org/pub/scm/linux/kernel/git/lee/mfd.git>
  - mmc: <https://git.kernel.org/pub/scm/linux/kernel/git/ulfh/mmc.git>
  - mtd: <https://git.kernel.org/pub/scm/linux/kernel/git/mtd/linux.git>
  - net: <https://git.kernel.org/pub/scm/linux/kernel/git/netdev/net.git>
  - of: <https://git.kernel.org/pub/scm/linux/kernel/git/robh/linux.git>
  - pci: <https://git.kernel.org/pub/scm/linux/kernel/git/pci/pci.git>
  - phy: <https://git.kernel.org/pub/scm/linux/kernel/git/phy/linux-phy.git>
  - pinctrl: <https://git.kernel.org/pub/scm/linux/kernel/git/linusw/linux-pinctrl.git>
  - platform:
    - chrome: <https://git.kernel.org/pub/scm/linux/kernel/git/chrome-platform/linux.git>
    - wmi, x86: <https://git.kernel.org/pub/scm/linux/kernel/git/pdx86/platform-drivers-x86.git>
  - pmdomain: <https://git.kernel.org/pub/scm/linux/kernel/git/ulfh/linux-pm.git>
  - power:
    - reset, supply: <https://git.kernel.org/pub/scm/linux/kernel/git/sre/linux-power-supply.git>
    - sequencing: <https://git.kernel.org/pub/scm/linux/kernel/git/brgl/linux.git>
  - pwm: <https://git.kernel.org/pub/scm/linux/kernel/git/ukleinek/linux.git>
  - regulator: <https://git.kernel.org/pub/scm/linux/kernel/git/broonie/regulator.git>
  - remoteproc, rpmsg: <https://git.kernel.org/pub/scm/linux/kernel/git/remoteproc/linux.git>
  - rtc: <https://git.kernel.org/pub/scm/linux/kernel/git/abelloni/linux.git>
  - scsi, ufs: <https://git.kernel.org/pub/scm/linux/kernel/git/jejb/scsi.git>
  - soundwire: <https://git.kernel.org/pub/scm/linux/kernel/git/vkoul/soundwire.git>
  - spi: <https://git.kernel.org/pub/scm/linux/kernel/git/broonie/spi.git>
  - spmi: <https://git.kernel.org/pub/scm/linux/kernel/git/sboyd/spmi.git>
  - thunderbolt, usb: <https://git.kernel.org/pub/scm/linux/kernel/git/gregkh/usb.git>
  - tty: <https://git.kernel.org/pub/scm/linux/kernel/git/gregkh/tty.git>
  - vfio: <https://github.com/awilliam/linux-vfio.git>
  - vhost, virtio: <https://git.kernel.org/pub/scm/linux/kernel/git/mst/vhost.git>
  - video:
    - backlight: <https://git.kernel.org/pub/scm/linux/kernel/git/lee/backlight.git>
    - fbdev: <https://git.kernel.org/pub/scm/linux/kernel/git/deller/linux-fbdev.git>
- fs
  - <https://git.kernel.org/pub/scm/linux/kernel/git/vfs/vfs.git>
  - btrfs: <https://git.kernel.org/pub/scm/linux/kernel/git/kdave/linux.git>
  - ext4: <https://git.kernel.org/pub/scm/linux/kernel/git/tytso/ext4.git>
- kernel
  - bpf: <https://git.kernel.org/pub/scm/linux/kernel/git/bpf/bpf-next.git>
  - cgroup: <https://git.kernel.org/pub/scm/linux/kernel/git/tj/cgroup.git>
  - dma: <https://git.kernel.org/pub/scm/linux/kernel/git/mszyprowski/linux.git>
  - futex, irq, locking, sched, time: <https://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git>
  - module: <https://git.kernel.org/pub/scm/linux/kernel/git/modules/linux.git>
  - power: <https://git.kernel.org/pub/scm/linux/kernel/git/rafael/linux-pm.git>
  - printk: <https://git.kernel.org/pub/scm/linux/kernel/git/printk/linux.git>
  - rcu: <https://git.kernel.org/pub/scm/linux/kernel/git/rcu/linux.git>
  - trace: <https://git.kernel.org/pub/scm/linux/kernel/git/trace/linux-trace.git>
  - workqueue: <https://git.kernel.org/pub/scm/linux/kernel/git/tj/wq.git>
- lib
  - crypto: <https://git.kernel.org/pub/scm/linux/kernel/git/ebiggers/linux.git>
- mm
  - <https://git.kernel.org/pub/scm/linux/kernel/git/akpm/mm.git>
  - memblock: <https://git.kernel.org/pub/scm/linux/kernel/git/rppt/memblock.git>
  - slab: <https://git.kernel.org/pub/scm/linux/kernel/git/vbabka/slab.git>
- net
  - <https://git.kernel.org/pub/scm/linux/kernel/git/netdev/net.git>
- security
  - landlock: <https://git.kernel.org/pub/scm/linux/kernel/git/mic/linux.git>
  - selinux: <https://git.kernel.org/pub/scm/linux/kernel/git/pcmoore/selinux.git>
- sound
  - <https://git.kernel.org/pub/scm/linux/kernel/git/tiwai/sound.git>
- virt
  - kvm: <https://git.kernel.org/pub/scm/virt/kvm/kvm.git>

## SoC

- <https://docs.kernel.org/process/maintainer-soc.html>
- downstream of
  - <https://git.kernel.org/pub/scm/linux/kernel/git/soc/soc.git>
  - <https://git.kernel.org/pub/scm/linux/kernel/git/clk/linux.git>
- mediatek: <https://git.kernel.org/pub/scm/linux/kernel/git/mediatek/linux.git>
- qcom: <https://git.kernel.org/pub/scm/linux/kernel/git/qcom/linux.git>
- rockchip: <https://git.kernel.org/pub/scm/linux/kernel/git/mmind/linux-rockchip.git>

## DRM Repos

- <https://gitlab.freedesktop.org/drm/kernel>
  - after `vX.(Y-1)-rc2` is cut, `drm-next` is reset and is open for dev
  - during the merge window, Dave Airlie closes `drm-next`, cuts
    `tags/drm-next-YYYY-MM-DD` from `drm-next`, and sends out the main pull
    request, `[git pull] drm for X.Y-rc1`
  - after each `vX.Y-rcZ`, `drm-fixes` is reset and is open for fixes
  - Dave Airlie cuts `tags/drm-fixes-YYYY-MM-DD` from `drm-fixes` and sends
    out `[git pull] drm fixes for X.Y-rc(Z+1)`
- misc: <https://gitlab.freedesktop.org/drm/misc/kernel>
  - Maxime Ripard sends out the main pull request, `[PULL] drm-misc-next`,
    to Dave Airlie
  - Smaller fixes, `[PULL] drm-misc-next-fixes`, follow
- amd: <https://gitlab.freedesktop.org/agd5f/linux>
  - Alex Deucher sends out the main pull request, `[pull] amdgpu, amdkfd drm-next-X.Y`,
    to Dave Airlie to pull from
  - Smaller fixes, `[pull] amdgpu drm-fixes-X.Y`, follow
- i915: <https://gitlab.freedesktop.org/drm/i915/kernel>
  - i915 is similar to xe
- mediatek: <https://git.kernel.org/pub/scm/linux/kernel/git/chunkuang.hu/linux.git>
- msm: <https://gitlab.freedesktop.org/drm/msm>
  - Rob Clark sends out the main pull request, `[pull] drm/msm: drm-msm-next-YYYY-MM-DD for vX.Y`
    to Dave Airlie
  - Smaller fixes, `[pull] drm/msm: drm-msm-fixes-YYYY-MM-DD for vX.Y-rcZ`, follow
- xe: <https://gitlab.freedesktop.org/drm/xe/kernel>
  - Intel sends out the main pull request, `[PULL] drm-xe-next`, to Dave
    Airlie
  - Smaller fixes, `[PULL] drm-xe-next-fixes`, follow
- <https://drm.pages.freedesktop.org/maintainer-tools/>
- drm
  - `drm/drm-next` branch is reset to `vX.Y-rc2` tag once the tag is created.
    The branch is then open for `Y+1` pull requests.
  - `drm/drm-fixes` branch is reset to `vX.Y-rcZ` tag once the tag is created.
    The branch is then open for `Y` fixes.
- drm-amd
  - `drm-amd/drm-next` branch is always open for `Y+1` changes
  - `drm-amd/amd-staging-drm-next` branch is always open for `Y` fixes
- drm-msm
  - `drm-msm/msm-next` branch is always open for `Y+1` changes.  It rebases
    against `drm/drm-next` occasionally.
  - `drm-msm/msm-fixes` branch is always open for `Y` fixes.  It rebases
    against `drm/drm-next` occasionally.
- drm-misc
  - `drm-misc/drm-misc-next` branch never resets/rebases, but merges in
    `vX.Y-rc1` to open for `Y+1` changes.  It occasionally merges in
    `vX.Y-rcZ` to resolve critical issues.
  - `drm-misc/drm-misc-fixes` branch resets to `vX.Y`, merges in

## b4

- <https://b4.docs.kernel.org/en/latest/installing.html>
  - `pip install b4`
- <https://b4.docs.kernel.org/en/latest/config.html>
- <https://b4.docs.kernel.org/en/latest/maintainer/overview.html>
  - `b4 mbox <msg-id>` saves all threads containing `<msg-id>` to a local mbox
    - it finds threads with `https://lore.kernel.org/all/<msg-id>/` by default
    - `-m <local-mbox>` finds in a local mbox instead
    - `-c` includes threads for later revisions as well
    - `-n <output-name>` specifies the output filename
  - `b4 am <msg-id>` is similar to `b4 mbox <msg-id>`, except it
    post-processes the threads for `git am`
    - `--no-cover` skips cover letter
  - `b4 diff <msg-id>` diffs against the prior revision
    - `-m <mbox1> <mbox2>` diffs against two local mboxes
  - `b4 dig -c <commit-id>` reverse looks up the msg id
    - `-a` looks up all revisions
- <https://b4.docs.kernel.org/en/latest/contributor/overview.html>
  - `b4 prep -n <topic>` creates `b4/<topic>` branch and creates an empty
    commit for the cover letter
    - it embeds a unique commit-id in the form of `<date>-<topic>-<hash>` in
      the cover letter
    - `-f <base-commit>` specifies the base commit for the branch
    - `-F <msg-id>` populates the new branch with patches from `<msg-id>`
      - if available, it also uses the base commit and the cover letter from
        the threads
    - `-e <branch>` preps an existing branch instead
  - `b4 prep` branch operations
    - `--edit-cover` edits the cover letter
    - `--auto-to-cc` runs `scripts/get_maintainer.pl` to add `To:`/`Cc:`
      to the cover letter
    - `--check` runs `scripts/checkpatch.pl` on all commits
    - `--cleanup br/<topic>` deletes everything related to the branch
  - `b4 send` one-time setup
    - `patatt genkey`
    - edit `.config/git/config` to add `[b4]` and `[patatt]`
    - `b4 send --web-auth-new`
    - `b4 send --web-auth-verify <token>`
  - `b4 send` sends the patches, tags the current revision, and increments the
    revision
    - `-o <tmpdir>` saves patches locally for double check
    - `--reflect` sends only to self for double check
  - `b4 trailers -u` retrieves trailers and updates commit messages
