# Trade Federation

## Prebuilts

- `Android.bp` automatically prefers prebuilt `java_import_host` modules over
  source `java_library_host` modules
- <https://android.googlesource.com/platform/tools/tradefederation/prebuilts/+/refs/heads/android17-release>
  provides prebuilts
  - this is done such that all branches use the same prebuilts
  - `tradefed` is prebuilt from `tools/tradefederation/core`
    - this is tradefed, to drive android automated testing
  - `compatibility-tradefed` is prebuilt from `test/suite_harness/common/host-side/tradefed`
    - this extends `tradefed` for xTS
  - `compatibility-host-util` is prebuilt from `test/suite_harness/common/host-side/util`
- <https://android.googlesource.com/platform/tools/deviceinfra/prebuilts/+/refs/heads/android17-release>
  - the cli interface of `cts-tradefed` is called console
  - `compatibility-tradefed` provides `com.android.compatibility.common.tradefed.command.CompatibilityConsole` console
  - but, by default, `cts-tradefed` uses closed-source ATS console from
    `ats_console_deploy` here
    - see `cts/tools/cts-tradefed/etc/cts-tradefed` and `USE_ATS`

## `AndroidTest.xml`

- tradefed loads test configs from `testcases/<module>/<module>.config`
  - `tools/tradefederation/core/src/com/android/tradefed/config/ConfigurationFactory.java`
  - during build, `AndroidTest.xml` is installed as `<module>.config`
- `<target_preparer>` prepares the taget
  - `com.android.tradefed.targetprep.suite.SuiteApkInstaller` installs an apk
  - `com.android.tradefed.targetprep.RootTargetPreparer` runs `adb root`
  - `com.android.tradefed.targetprep.RunCommandTargetPreparer` runs shell cmds
  - `com.android.tradefed.targetprep.PushFilePreparer` pushes files
  - `com.android.tradefed.targetprep.DeviceSetup` keeps screen on, etc.
- `<test>` specifies a test runner
  - `com.android.tradefed.testtype.AndroidJUnitTest` runs a junit-based test
    - `AndroidJUnitTest` is host side
    - `AndroidJUnitRunner` is device side
  - `com.android.tradefed.testtype.GTest` runs a gtest-based binary
  - `com.android.tradefed.testtype.HostTest` runs a host-side test

## Filtering

- `run commandAndExit cts-dev` loads and runs all test modules
  - `commandAndExit` tells tradefed to exit after completion
  - `cts-dev` tells tradefed to load built-in `cts-dev.xml` config
  - imagine all module configs are loaded to build a huge in-memory config
- `run ... <module>:<class>#<method>` runs a specific test of a module
  - it is the same as `-m <module> -t <class>#<method>`
  - it is a syntax sugar for `--include-filter "<module> <class>#<method>"`
  - tradefed sees `<module>` and loads only the specific module config
  - tradefed forwards `<class>#<method>` to the module test runner
    - e.g., `AndroidJUnitTest` test runner invokes `am instrument` and
      translates the filter to `-e class <class>#<method>`
- wildcards
  - `<module>` must be exact match
  - `<class>` is actually `[<package>.]<class>`
    - `<package>` must be exact match if specified
    - `<class>` depends on runner but can expect full wildcard support
      - but if `<method>` is specified, `<class>` must be exact match
  - `<method>` depends on runner but can expect full wildcard support
- `--include-filter "<module> <class>#<method>"` internal
  - `BaseTestSuite` parses it into `mIncludeFilters` and passes it to
    `SuiteModuleLoader`
  - `SuiteModuleLoader`
    - `loadConfigsFromDirectory` and `shouldRunModule` use `<module>` to
      decide which configs to load
    - `addFiltersToTest -> addTestIncludes` passes `<class>#<method>` to
      `ITestFilterReceiver::addIncludeFilter`
  - `AndroidJUnitTest` is a `ITestFilterReceiver` and translates the filter to
    - if `isRegex`, `tests_regex`
    - if `isClassOrMethod`, `class`
    - else, `package`
  - `RemoteAndroidTestRunner` runs the test
    - `getAmInstrumentCommand` builds `am instrument -w ...`
  - `DeqpTestRunner` is both a `ITestFilterReceiver` and a runner
    - `filterTests` applies the test filter to trim down the test list
