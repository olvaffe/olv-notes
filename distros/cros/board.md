Chrome OS Board
===============

## General

- <https://dev.gentoo.org/~zmedico/portage/doc/man/make.conf.5.html>
  - `FEATURES`
    - for features of portage itself
  - `USE`
    - for optional features of packages and their dependencies
- <https://dev.gentoo.org/~zmedico/portage/doc/man/portage.5.html>
  - `package.mask`
    - mask out packages from being installed
  - `package.unmask`
    - unmask packages
  - `package.use`
    - set/unset USE flags for packages
  - `package.use.mask`
    - mask out or unmask USE flags for packages
  - `use.mask`
    - mask out or unmask USE flags globally

## amd64-generic

- profile inheritance tree
  - ordering is depth-first, left-to-right
  - `overlays/overlay-amd64-generic/profiles/base/parent`
    - `third_party/chromiumos-overlay/profiles/default/linux/amd64/10.0/chromeos/parent`
      - `third_party/chromiumos-overlay/profiles/default/linux/amd64/10.0/parent`
        - `third_party/chromiumos-overlay/profiles/default/linux/amd64/parent`
          - `third_party/chromiumos-overlay/profiles/base`
          - `third_party/chromiumos-overlay/profiles/default/linux`
          - `third_party/chromiumos-overlay/profiles/arch/amd64/parent`
            - `third_party/chromiumos-overlay/profiles/arch/base`
            - `third_party/chromiumos-overlay/profiles/features/multilib/lib32/parent`
              - `third_party/chromiumos-overlay/profiles/features/multilib`
        - `third_party/chromiumos-overlay/profiles/releases/10.0/parent`
          - `third_party/chromiumos-overlay/profiles/releases`
      - `third_party/chromiumos-overlay/profiles/targets/chromeos`
      - `third_party/chromiumos-overlay/profiles/features/llvm/amd64/parent`
        - `third_party/chromiumos-overlay/profiles/features/llvm`
    - `third_party/chromiumos-overlay/profiles/features/selinux`
- `setup_board -b amd64-generic --profile fuzzer`
  - `overlays/overlay-amd64-generic/profiles/fuzzer/parent`
    - `overlays/overlay-amd64-generic/profiles/base/parent`
      - see above
    - `third_party/chromiumos-overlay/profiles/features/sanitizers/fuzzer/asan/amd64/parent`
      - `third_party/chromiumos-overlay/profiles/features/sanitizers/asan/amd64/parent`
        - `third_party/chromiumos-overlay/profiles/features/sanitizers/asan/parent`
          - `third_party/chromiumos-overlay/profiles/features/sanitizers`
      - `third_party/chromiumos-overlay/profiles/features/sanitizers/fuzzer/asan/parent`
        - `third_party/chromiumos-overlay/profiles/features/sanitizers/fuzzer`
- `./build_packages` builds these packages by default
  - `virtual/target-os`
    - this depends on `virtual/target-chromium-os` which includes a long list
      of packages
  - `virtual/target-os-dev`
  - `virtual/target-os-factory`
  - `virtual/target-os-factory-shim`
  - `virtual/target-os-test`
  - `chromeos-base/autotest-all`
- these packages depend on `virtual/opengles` when `USE=opengles`
  - `chromeos-base/chromeos-chrome`
  - `chromeos-base/drm-tests`
  - `chromeos-base/glbench`
  - `dev-util/apitrace`
  - `media-gfx/deqp`
  - `media-libs/libepoxy`
  - `media-libs/waffle`
  - maybe more
- these packages depend on `media-libs/vulkan-loader` when `USE=vulkan`
  - `chromeos-base/drm-tests`
  - `chromeos-base/vkbench`
  - `dev-util/vulkan-tools`
  - `media-libs/virglrenderer`
  - `virtual/vulkan-icd`
  - maybe more
- these packages depend on `virtual/vulkan-icd` when `USE=vulkan`
  - `chromeos-base/drm-tests`
  - `chromeos-base/vkbench`
  - `media-gfx/deqp`
  - maybe more
- virtualization
  - `USE=kvm_host` means enable KVM support in the host kernel
    - `target-chromium-os` uses the flag to add `crostini_client`,
      `vm_host_tools`, and `termina-dlc`
    - `vm_host_tools` depends on `crosvm`
  - when `USE=crosvm-gpu`, `crosvm` depends on `virglrenderer`
  - `USE=kvm_guest` means building for the (crostini) container image
