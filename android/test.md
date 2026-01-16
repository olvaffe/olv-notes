Android Test
============

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

## Harness Prebuilts

- `Android.bp` automatically prefers prebuilt `java_import_host` modules over
  source `java_library_host` modules
- <https://android.googlesource.com/platform/tools/tradefederation/prebuilts/+/refs/heads/android16-qpr2-release>
  provides prebuilts
  - this is done such that all branches use the same prebuilts
  - `tradefed` is prebuilt from `tools/tradefederation/core`
    - this is tradefed, to drive android automated testing
  - `compatibility-tradefed` is prebuilt from `test/suite_harness/common/host-side/tradefed`
    - this extends `tradefed` for xTS
  - `compatibility-host-util` is prebuilt from `test/suite_harness/common/host-side/util`
- <https://android.googlesource.com/platform/tools/deviceinfra/prebuilts/+/refs/heads/android16-qpr2-release>
  - the cli interface of `cts-tradefed` is called console
  - `compatibility-tradefed` provides `com.android.compatibility.common.tradefed.command.CompatibilityConsole` console
  - but, by default, `cts-tradefed` uses closed-source ATS console from
    `ats_console_deploy` here
  - see `cts/tools/cts-tradefed/etc/cts-tradefed`

## VTS

- <https://android.googlesource.com/platform/test/vts/+/refs/heads/android16-qpr2-release/>
- <https://android.googlesource.com/platform/test/vts/+/refs/heads/android16-qpr2-release/tools/vts-core-tradefed/Android.bp>
  - `compatibility_test_suite_package` defines `vts` test suite
    - i guess this allows various tests to specify `test_suites: ["vts"]`,
      which copies the tests to `android-vts` for distribution
  - `tradefed_binary_host` defines `vts-tradefed` binary
