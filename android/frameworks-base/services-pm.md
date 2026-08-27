# Package Manager

## Users

- <https://source.android.com/docs/devices/admin/multi-user>
- `UserHandle` defines several constants
  - `USER_SYSTEM` is 0
  - `MIN_SECONDARY_USER_ID` is 10
- `adb shell am`
  - before login, `get-current-user` returns 0
  - `switch-user 10` logs in user 10
- `adb shell cmd user`
  - `list` lists all users
  - `is-headless-system-user-mode` returns true
  - starting an activity before login fails
    - `Activity not started, not allowed for the given user.`
    - `adb shell cmd user activities-allowlist android.os.usertype.system.HEADLESS disable` works around the limit

## Permissions

- app must declare permissions in manifest first
  - with `<uses-permission>`
- normal permissions
  - these are safe permissions such as `INTERNET`
  - granted automatically at install and cannot be revoked
- runtime permissions
  - these are dangerous permissions such as `CAMERA`
  - app must request at runtime; user can allow/deny
  - `pm grant` only works for runtime permissions
- special permissions
  - these are most dangerous permissions such as `MANAGE_EXTERNAL_STORAGE`
  - app must direct users to settings at runtime; user can allow/deny
  - `appops set` only works for special permissions
- external storage
  - `READ_EXTERNAL_STORAGE` is a deprecated runtime permission
    - since android 10, it allows reading media files on external storage
    - before android 10, it allows reading any files on external storage
  - `WRITE_EXTERNAL_STORAGE` is a deprecated runtime permission
    - since android 10, it is ignored; any app can add media files on external storage
    - before android 10, it allows writing any files on external storage
  - `MANAGE_EXTERNAL_STORAGE` is a special permission
    - it allows full access to external storage
    - only available since android 11
  - `android:requestLegacyExternalStorage`
    - when app target sdk is less than 29 (android 10), this flag is implied
    - it restores the behavior prior to android 10
  - private app dir on external storage
    - since android 4.4, app can read/write its private app dir on external
      storage without any permission
    - path under app mnt namespace: `/sdcard/Android/data/<pkg>`
    - path under root namespace: `/data/media/10/Android/data/<pkg>`
    - for comparision, path of private app dir on internal storage is
      `/data/user/10/<pkg>`
