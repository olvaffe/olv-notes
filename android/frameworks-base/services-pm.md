Package Manager
===============

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
