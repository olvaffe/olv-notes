# Android SELinux

## App Data

- `mkdir -p /data/my-data`
- `chmod 777 /data/my-data`
- `chcon u:object_r:app_data_file:s0 /data/my-data`
