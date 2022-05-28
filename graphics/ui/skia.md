Skia
====

## Build and Use

- <https://skia.org/docs/user/build/>
  - `git clone https://skia.googlesource.com/skia.git`
  - `cd skia`
  - `./tools/git-sync-deps`
  - `./tools/install_dependencies.sh`
  - `./bin/gn gen out --args='is_official_build=false'`
  - `ninja -C out`
- args
  - list
    - `gn args --list out`
    - `gn/BUILDCONFIG.gn`
    - `gn/skia.gni`
  - vulkan
    - `skia_use_x11 = false`
    - `skia_use_gl = false`
    - `skia_use_egl = false`
    - `skia_use_vulkan = true`
  - disable font
    - `skia_use_freetype = false`
    - `skia_use_fontconfig = false`
    - `skia_use_harfbuzz = false`
- terms
  - gm stands for golden master test suite
  - dm stands for dungeon master and runs unit tests and gms
    - first commit says "dm is like gm, but faster and with fewer features"
  - fm was an attemp to become better dm?
    - first commit says "FM, a dumber new testing tool"
- <https://skia.org/docs/dev/testing/>
  - `./out/fm --resourcePath resources --sources arithmode --backend vk`
    - `--listTests`
    - `--listGMs`
  - `./out/skqp platform_tools/android/apps/skqp/src/main/assets skqp/rendertests.txt results`
    - `./out/skqp NOT USED results '^vk_arithmode$'` seems better
  - `./out/dm --resourcePath resources --src gm --config vk --verbose`
- skia on angle
  - <https://skia.org/docs/user/special/angle/>
- skia on vulkan
  - <https://skia.org/docs/user/special/vulkan/>
- skia on android
  - <https://skia.org/docs/user/build/#android>
  - <https://skia.googlesource.com/skia/+/main/tools/skqp/README.md>
