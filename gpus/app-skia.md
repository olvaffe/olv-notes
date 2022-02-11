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
- tests
  - `./out/fm --resourcePath resources --sources arithmode --backend vk`
    - `--listTests`
    - `--listGMs`
  - `./out/skqp platform_tools/android/apps/skqp/src/main/assets skqp/rendertests.txt results`
    - `./out/skqp NOT USED results '^vk_arithmode$'` seems better
  - `./out/dm --resourcePath resources --src gm --config vk --verbose`
