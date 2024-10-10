Armbian
=======

## Build Flow

- `compile.sh` calls `cli_entrypoint`
  - if no cmd is specified, `ARMBIAN_COMMAND="undecided"` initially
  - `cli_undecided_pre_run` changes the cmd to `build`
  - `ARMBIAN_COMMANDS_TO_HANDLERS_DICT` maps `build` to `standard_build`
- `cli_standard_build_run`
  - `prep_conf_main_build_single` prepares the configs
    - `interactive_config_ask_board_list` sets `BOARD`
    - `interactive_config_ask_branch` sets `BRANCH`
    - `interactive_config_ask_release` sets `RELEASE`
    - `interactive_config_ask_desktop_build` and
      `interactive_config_ask_standard_or_minimal` sets `BUILD_DESKTOP` and
      `BUILD_MINIMAL`, and `SELECTED_CONFIGURATION`
    - `config_pre_main` sanitizes the configuration
    - `do_main_configuration` does the configuration
    - `do_extra_configuration` does more configuration
  - `full_build_packages_rootfs_and_image`
    - `main_default_build_packages`
      - `determine_artifacts_to_build_for_image` determines what to build
        - e.g., `uboot`, `kernel`, `firmware`, `rootfs`, and `armbian-*`
      - `build_artifact_for_image` builds one artifact at a time
    - `build_rootfs_and_image`
      - `get_or_create_rootfs_cache_chroot_sdcard` builds `rootfs` artifact
      - `install_distribution_specific` and `install_distribution_agnostic`
        tweak the rootfs
        - this also installs `linux-image` and `linux-dtb` deb packages
      - `create_sources_list_and_deploy_repo_key` generates
        `/etc/apt/sources.list`
      - `prepare_partitions` creates (`truncate`) and partitions (`sfdisk`)
        the disk image, loop-mounts the disk image, and tweaks the rootfs more
      - `create_image_from_sdcard_rootfs` copies files from rootfs into the
        disk image
      - `output_images_compress_and_checksum` compresses the disk image
- `build_artifact_for_image` builds artifacts
  - `artifact-uboot.sh` builds uboot and packs a deb package
  - `artifact-kernel.sh` builds kernel and packs a deb package
  - `artifact-firmware.sh` packs a firmware deb package
    - <https://github.com/armbian/firmware>
  - `artifact-rootfs.sh` bootstraps the rootfs
    - `create_new_rootfs_cache` runs `debootstrap`
