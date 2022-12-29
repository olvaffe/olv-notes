Chrome OS DLC
=============

## DLC

- <https://chromium.googlesource.com/chromiumos/platform2/+/HEAD/dlcservice/docs/developer.md>
- `USE=dlc` to include `dlcservice` and `dlcservice-client` packages in the
  build
- `dlcservice` is a daemon providing D-Bus api to chrome and other system
  daemons
  - it can be requested to download and mount DLCs 
- a DLC package can be created by inheriting `dlc.eclass` in ebuild

## Paths

- const paths
  - `const char kDlcManifestRootpath[] = "/opt/google/dlc/";`
    - The root path of all DLC module manifests.
  - `const char kDlcImageRootpath[] = "/var/cache/dlc/";`
    - The root path of all DLC module images.
  - `constexpr char kDlcPreloadedImageRootpath[] = "/var/cache/dlc-images";`
    - these are DLCs shipped with the system image
  - `constexpr char kDlcServicePrefsPath[] = "/var/lib/dlcservice";`
  - `constexpr char kUsersPath[] = "/home/user";`
- `DlcManager::Initialize` creates a `DlcBase` for each directory under
  `/opt/google/dlc`
- `DlcBase::Initialize`
  - `content_dir` is `/var/cache/dlc`
  - `prefs_path_` is `/var/lib/dlcservice`
- when `dlcservice` is requested to install a DLC,
  - `DlcService::Install` calls `DlcManager::Install` which calls
    `DlcBase::Install`
    - `DlcBase::CreateDlc` creates the directory structure under
      `/var/cache/dlc/<name>`
    - `DlcBase::PreloadedCopier` copies from `/var/cache/dlc-images` to
      `/var/cache/dlc`, if the dlc is preloaded
    - otherwise, `DlcService::InstallWithUpdateEngine` requests update engine
      to download the DLC
  - `DlcService::FinishInstall` calls `DlcManager::FinishInstall` which calls
    `DlcBase::FinishInstall`
    - `DlcBase::Mount` requests image loader to mount dlc under
      `/run/imageloader`
- DLCs are mounted at `/run/imageloader/<name>/package`
  - it is the mountpoint of a dm device
    - verified with `findmnt`
  - the dm device targets a loop device
    - verified with `dmsetup ls --tree`
  - the loop device is set up from `/var/cache/dlc/<name>/package/dlc_a/dlc.img`
    - verified with `losetup`
  - `/var` is a bind mount of `/mnt/stateful_partition/encrypted/var`
  - `/mnt/stateful_partition/encrypted` is a mount point of
    `/dev/mapper/encstateful`
  - `/dev/mapper/encstateful` targets `/dev/loop0`
  - `/dev/loop0` is `/mnt/stateful_partition/encrypted.block`
