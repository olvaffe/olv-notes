Platform2 libsegmentation
=========================

## override device info

- `echo -n CAIQAh== > /run/libsegmentation/feature_device_info`
  - `kTempDeviceInfoPath` points to the path for temporary override
  - `FeatureManagementUtil::ReadDeviceInfo` base64-decodes the string and
    parses it as `DeviceInfo` proto
- `echo -n CAIQAh== | base64 -d | protoc --decode=libsegmentation.DeviceInfo protos/device_info.proto`
  - `feature_level: FEATURE_LEVEL_1`
  - `scope_level: SCOPE_LEVEL_1`
- `feature_explorer --feature_level` prints 1
- `feature_explorer --scope_level` prints 1
