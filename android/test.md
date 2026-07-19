# Android Test

## Suite Harness

- this is the framework for xTS, built on top of `tools/tradefederation`
- `compatibility-host-util` is used by various host-side tests
  - e.g., `com.android.compatibility.common.util.PropertyUtil` can be used to
    query device properties
- `compatibility-tradefed` is used by various `xts-tradefed`
  - it extends `tools/tradefederation`
  - `CompatibilitySuiteModuleLoader` extends `SuiteModuleLoader`
  - `CompatibilityTestSuite` extends `BaseTestSuite`
- e.g., `cts-tradefed` uses `compatibility-tradefed`
  - there are two twists though
  - firstly, to ensure all branches use the same version of suite harness, it
    does not build the test harness from source but uses prebuilts from
    `tools/tradefederation/prebuilts/test_harness`
  - secondly, `cts-tradefed` is actually a script that defaults to use
    closed-source ATS harness from
    `tools/deviceinfra/prebuilts/ats_console_deploy.jar` instead
    - <https://source.android.com/docs/core/tests/development/android-test-station/ats-user-guide>

## `Android.bp`

- `cc_test` builds a native test and `android_test` builds a test apk
  - the artifacts (binary, config, extra data) are copied to
    `$ANDROID_PRODUCT_OUT/testcases/<test>`
  - if native, the binary is also copied to
    `$ANDROID_PRODUCT_OUT/data/nativetest64` for distribution
- `java_test_host` builds a host test
  - the artifacts (jar, config) are copied to
    `$ANDROID_HOST_OUT/testcases/<test>`
  - the host test typically uses adb to communicate with dut
- options
  - `test_suites` specifies the suites
    - the artifacts are also copied to
      `$ANDROID_HOST_OUT/<suite>/android-<suite>/testcases/<test>`
      for distribution
  - `test_config` specifies a customized config for tradefed
    - otherwise, soong generates a default one
  - `data` specifies extra data
    - they can refer to native tests to include the native test binaries
  - `auto_gen_config: false`
    - this is often used with `test_options: { extra_test_configs: { ... } }`
    - `atest <test>` will say that there is no test to run
      - use `atest <config>` instead
- `atest <test>`
  - it builds the test and its harness (tradefed)
  - it invokes `atest_tradefed.sh` to run the test
    - installs apk if any
    - pushes test binaries/data according to the config if any
    - runs the test
    - removes test binaries/data and apk

## VTS

- <https://android.googlesource.com/platform/test/vts/+/refs/heads/android17-release/>
- <https://android.googlesource.com/platform/test/vts/+/refs/heads/android17-release/tools/vts-core-tradefed/Android.bp>
  - `compatibility_test_suite_package` defines `vts` test suite
    - i guess this allows various tests to specify `test_suites: ["vts"]`,
      which copies the tests to `android-vts` for distribution
  - `tradefed_binary_host` defines `vts-tradefed` binary
