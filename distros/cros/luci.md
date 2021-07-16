Chrome OS CI
============

## CI

- <https://ci.chromium.org/>
- projects
  - chromium <https://ci.chromium.org/p/chromium>
  - fuchsia <https://ci.chromium.org/p/fuchsia>
  - angle <https://ci.chromium.org/p/angle>
- builders
  - each project can have many builders
  - angle's android-arm64-rel
    <https://ci.chromium.org/p/angle/builders/ci/android-arm64-rel>
  - fuchsia's clang-linux-arm64
    <https://ci.chromium.org/p/fuchsia/builders/toolchain.ci/clang-linux-arm64>
  - chromium's Linux Builder
    <https://ci.chromium.org/p/chromium/builders/ci/Linux%20Builder>
- builds
  - each builder can have many builds
  - build 140196 of chromium's Linux Builder
    <https://ci.chromium.org/ui/p/chromium/builders/ci/Linux%20Builder/140196/overview>
- groups
  - a group groups related builders together
  - a group can present a console view
  - a builder can belong to multiple groups
