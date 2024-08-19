Platform2 ARC
=============

## `session_manager`

- after `session_manager` starts chrome, chrome requests "StartArcMiniContainer"
  - request handled by `session_manager` again
  - `session_manager` sends upstart "start-arc-instance" impulse
  - it also calls `run_oci start`, which starts an OCI container
  - it reads container /init pid from `/run/containers/android-run_oci`
- chrome requests "UpgradeArcContainer" after user logs in
  - `session_manager` sends upstart "continue-arc-boot" impulse
- once chrome considers ARC booted, it requests "EmitArcBooted"
  - `session_manager` sends upstart "arc-booted" impulse
- when "StopArcInstance" is requested (or when container /init crashed)
  - `session_manager` requests container /init to exit
  - after /init exits, it calls `run_oci destroy` to shutdown the container
  - it sends upstart "stop-arc-instance" impulse

## `run_oci` and arc-setup

- when `run_oci start` is called, these hooks are invoked
  - precreate
  - prechroot
  - prestart
  - <start container and run /init>
  - poststart
- when `run_oci destroy` is called, these hooks are invoked
  - poststop
- `run_oci` parses "<container-bundle>/config.json" for hooks
  - precreate maps to "arc-setup --mode=setup"
  - prechroot maps to "arc-setup --mode=pre-chroot"
- arc-setup parses "/usr/share/arc-setup/config.json" for configs
- arc-setup has these modes
  - triggered by `run_oci start`
    - setup, setup mountpoints and etc.
    - pre-chroot, more mountpoints and write some files
  - triggerd by `start-arc-instance`
    - read-ahead
  - triggerd by "continue-arc-boot"
    - boot-continue, set up /data parition, maybe start adbd,  run
      "/system/bin/arcbootcontinue" inside Android, start arc-sdcard, etc.
    - mount-sdcard, indirectly triggered to mount sdcard
  - triggerd by "arc-booted"
    - update-restorecon-last
  - triggerd by "stop-arc-instance"
    - stop

## Development

- Logs can be found at /var/log/arc.log
- "android-sh" gives you shell
  - it gets container pid and root from `/run/containers/android-run_oci/container.pid`
  - it executes "nsenter" to enter the name space
  - it uses "runcon" to start shell in the desired context

## adb

- enable adb in settings
  - or, `android-sh -c "start adbd"`
  - or, `android-sh -c "setprop persist.sys.usb.config adb"`
- `adb connect 100.115.92.2:5555`
- public key auth
  - `adb keygen adbkey`
  - `cat ~/.android/adbkey.pub | android-sh -c "cat > /data/misc/adb/adb_keys"`
  - `android-sh -c "restorecon /data/misc/adb/adb_keys"`
- DUT with test image runs `sslh` to redirect adb packets to DUT:22 to 100.115.92.2:5555
  - host can `adb connect <DUT-IP>:22`

## Cross-Compile for ARC++ P

- `FEATURES="noclean" emerge arc-foo` and find the generated meson cross files
  from under `/build/$board/tmp/portage` as usual
  - requires `SYSROOT=/build/$board` as usual
  - the pkg-config wrapper additionally requires `ARC_SYSROOT` and `ABI`
    - <https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/master/eclass/arc-build-constants.eclass>
    - <https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/master/eclass/arc-build.eclass>
    - `ARC_SYSROOT=/build/$BOARD/opt/google/containers/android`
    - `ABI=arm64`
- links
  - <https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/refs/heads/main/sys-devel/arc-toolchain-p/>
    - tarball has toolchains and sysroots for x86 and arm, both 32-bit and 64-bit
  - <https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/refs/heads/main/sys-devel/arc-build/>
    - .pc files for use with mesa
  - <https://mesonbuild.com/Cross-compilation.html>

## Cross-Compile for ARCVM R

- mesa
  - `SYSROOT=/build/$BOARD \
     ARC_SYSROOT=/build/$BOARD/opt/google/vms/android \
     ABI=arm64 \
     meson --cross-file meson.aarch64-linux-android.arm64.ini out-$BOARD \
     -Dgallium-drivers= -Dvulkan-drivers=virtio-experimental \
     -Dbuildtype=debug -Dplatforms=android -Dplatform-sdk-version=30`
  - for freedreno,

     -Degl=enabled
     -Dllvm=disabled
     -Dgallium-xa=false
     -Dgbm=false
     -Ddri-search-path=/vendor/lib64/dri
     -Dgles-lib-suffix=_mesa
     -Degl-lib-suffix=_mesa
     -Dplatforms=android
     -Dplatform-sdk-version=30
     -Dgallium-drivers=freedreno
     -Dfreedreno-virtio=true
     -Dvulkan-drivers=
  - and push `out-$BOARD/src/egl/libEGL_mesa.so` to
    `/vendor/lib64/egl/libEGL_mesa.so` and push
    `out-$BOARD/src/gallium/targets/dri/msm_dri.so` as
    `/vendor/lib64/dri/virtio_gpu_dri.so`
- kernel
  - `cd src/third_party/kernel/v5.10-arcvm`
  - `make ARCH=arm64 O=ARCVM CROSS_COMPILE=aarch64-cros-linux-gnu- arm64_arcvm_defconfig`
  - `scp -C ARCVM/arch/arm64/boot/Image dut:/opt/google/vms/android/vmlinux`

## D-Bus

- for some projects under platform2, there is a `dbus_bindings` subdirectory
  - e.g., `login_manager/dbus_bindings/org.chromium.SessionManagerInterface.xml`
- for others, there is `system_api/dbus`
  - e.g., `system_api/dbus/vm_concierge`
  - these are a part of "system api" and are used by chrome the browser as
    well
- to get user hash,
  - `dbus-send --system --dest=org.chromium.SessionManager --print-reply \
      /org/chromium/SessionManager org.chromium.SessionManagerInterface.RetrievePrimarySession`
  - `awk -F\" 'NR==3{print $2}'`
- to get `vm_concierge` pid,
  - `dbus-send --system --dest=org.freedesktop.DBus --print-reply \
      / org.freedesktop.DBus.GetConnectionUnixProcessID string:org.chromium.VmConcierge`
- to list vms
  - `concierge_client --cryptohome_id=<hash> --list_vms`
- to get arcvm cid,
  - `concierge_client --cryptohome_id=<hash> --name=arcvm --get_vm_cid`
  - should work for `termina` and `borealis` as well

## Modify Vendor Image

- container method 1
  - <https://docs.mesa3d.org/android.html#replacing-android-drivers-on-chrome-os>
  - `cd /opt/google/containers/android`
  - `mkdir tmp`
  - `mount vendor.raw.img tmp`
  - `cp -a tmp vendor`
    - do exactly this to make sure `vendor` has the right selinux context
    - `ls -lZ` to double-chek
  - `umount tmp`
  - `rmdir tmp`
  - edit `config.json` to modify `/vendor` mountpoint
    - `type` is `bind`
    - `source` is `vendor`
    - `options` should have `bind` and `rw`
  - `restart ui`
  - modifications to `/opt/google/containers/android/vendor` will be reflected
    in the container
- container method 2
  - `scp dut:/opt/google/containers/android/vendor.raw.img .`
  - `scp dut:/etc/selinux/arc/contexts/files/android_file_contexts .`
  - `mkdir chroot`
  - `sudo unsquashfs -no-xattrs -d chroot/vendor vendor.raw.img`
  - modify `chroot/vendor`
  - `rm -f repack.img`
  - `sudo mksquashfs chroot/vendor repack.img -comp gzip -context-file android_file_contexts -mount-point /vendor`
  - `scp repack.img dut:/opt/google/containers/android/vendor.raw.img`
- vm method 1
  - use userdebug image
  - `adb root`
  - overlayfs
  - this is good until vm reboot/crash
- vm method 2
  - `scp dut:/opt/google/vms/android/vendor.raw.img .`
  - `scp dut:/etc/selinux/arc/contexts/files/android_file_contexts_vm .`
  - `mkdir chroot`
  - `sudo unsquashfs -no-xattrs -d chroot/vendor vendor.raw.img`
  - modify `chroot/vendor`
  - `rm -f repack.img`
  - `sudo mksquashfs chroot/vendor repack.img -comp lz4 -Xhc -b 256K -context-file android_file_contexts_vm -mount-point /vendor`
  - `scp repack.img dut:/opt/google/vms/android/vendor.raw.img`

## libvda

- crosvm uses libvda to emulate a virtio-video device
  - `register_video_device` registers the device
  - backend can be configured by `--video-decoder` and `--video-encoder`
  - most boards use the legacy `libvda` while some use the newer `libvda-vd`
    - `USE=crosvm-virtio-video-vd`
- libvda talks to chrome using
  - the legacy `GpuArcVideoDecodeAccelerator` mojom interface, or
  - the newer `GpuArcVideoDecoder` mojom interface
- `LibvdaGpuTest.DecodeFileGpu` test
  - `initialize` initializes
    - `arc::VafConnection::Get` gets a `VideoAcceleratorFactory` connection
    - `arc::GpuVdaImpl::Create` returns (older) `GpuVdaImpl`
  - `init_decode_session` initializes a decode session using the specified
    codec profile
  - `get_vda_capabilities` returns the supported input/output formats
  - `arc::test::DecodeEventThread` starts a decode thread
    - it creates a gbm device
    - it polls `event_pipe_fd` of the session
    - on `PROVIDE_PICTURE_BUFFERS` event,
      - clears previously allocates gbm bos
      - `vda_set_output_buffer_count` sets the output buffer count
      - `gbm_bo_create` allocates the minimum required number of bos
      - `vda_use_output_buffer` passes the bos to vda
    - on `PICTURE_READY` event,
      - `vda_reuse_output_buffer` reuses one of the gbm bos
  - `base::WritableSharedMemoryRegion` allocates a shmem for the encoded data
  - `vda_decode` passes the shmem to vda for decoding
- chrome implementations
  - `OOPArcVideoAcceleratorFactory` implements `VideoAcceleratorFactory` mojom
  - `GpuArcVideoDecodeAccelerator` implements `GpuArcVideoDecodeAccelerator`
    mojom
    - this uses `media::VdVideoDecodeAccelerator::Create`, VD-backed VDA
    - `GpuArcVideoDecoder::Decode` is called when libvda provides more
      encoded data
      - the encoded data is stored in an shmem
      - the shmem is wrapped in a `base::UnsafeSharedMemoryRegion` then in a
        `media::BitstreamBuffer`
      - `vda->Decode` is called
    - `GpuArcVideoDecodeAccelerator::ProvidePictureBuffersWithVisibleRect` is
      called when media needs buffers
      - it calls `client_->ProvidePictureBuffers`
      - libvda translates that to `PROVIDE_PICTURE_BUFFERS`
    - `GpuArcVideoDecodeAccelerator::ImportBufferForPicture` is called when
      libvda provides dma-bufs in response to `ProvidePictureBuffers`
      - the dma-buf is wrapped in a `gfx::GpuMemoryBufferHandle`
      - `vda_->ImportBufferForPicture` is called
    - `GpuArcVideoDecodeAccelerator::PictureReady` is
      called when a frame has been decoded
      - it calls `client_->PictureReady`
      - libvda translates that to `PICTURE_READY`
    - `GpuArcVideoDecodeAccelerator::ReusePictureBuffer` is called when libvda
      has consumed the decoded frame
      - i guess it passes gbm bo ownership back to vda
      - `vda_->ReusePictureBuffer` is called
  - `GpuArcVideoDecoder` implements `GpuArcVideoDecoder` mojom
    - this uses `media::VideoDecoderPipeline::Create`, VD
