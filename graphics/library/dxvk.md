# DXVK

## Native Build

- build
  - `apt install libglfw3-dev` (or `libsdl3-dev`)
  - `git clone --recurse-submodules https://github.com/doitsujin/dxvk.git`
  - `meson setup out -Dprefix=$PWD/out/install -Dbuildtype=debug`
- no built-in test nor demo
- `DXVK_WSI_DRIVER=GLFW` or `SDL3`
