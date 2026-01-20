glTF
====

## Specs

- <https://registry.khronos.org/glTF/>
  - <https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html>
- <https://registry.khronos.org/KTX/>

## What is glTF?

- a JSON file that describes the structure and composition of a scene
  containing 3D models
- it is possible to pack the JSON file and the binary data of buffers/images
  into a single file

## Top-level elements

- se `glTF 2.0 Quick Reference Guide`
- `scenes` is an array of scenes
  - each scene has a name and an array of nodes
- `nodes` is an array of nodes
  - each node is either a mesh or a camera
  - there is also a per-node local transformation
- `cameras` is an array of cameras
  - each camera has a name and some attributes (position, fov, near/far, etc.)
- `meshes` is an array of meshes
  - each mesh has a name and an array of primitives
  - each primitive is a list of triangles whose attributes and indices are
    accessors and has an material
- `accessors` is an array of accessors
  - each accessor specifies the interpretation (float/int, vec[234], count) of
    a bufferView
- `bufferViews` is an array of bufferViews
  - each bufferView is a (offset, size) area of a buffer
- `buffers` is an array of buffers
  - each buffer is an URI referring the contents
- `materials` is an array of materials
  - each material specifies the PBR parameters as well as textures
- `textures` is an array of textures
  - each texture refers an image and a sampler
- `images` is an array of images
  - each image is an URI referring the contents
- `samplers` is an array of samplers
  - each sampler specifies the minFilter, magFilter, wrap, etc.
- `skins` is an array of skins
- `animations` is an array of animations

## Scene Graph

- each scene has an array of nodes
- each node can have an array of children nodes
  - this allows glTF to model a scene hierarchy

## KTX

- Khronos Texture, KTX
- <https://www.khronos.org/assets/uploads/apis/KTX-2.0-Launch-Overview-Apr21_.pdf>
  - GPU Compressed Formats
    - BC1/BC3/BC7
      - Intel, AMD, nVidia (desktop and tegra), Apple (Mx)
    - ETC1/ETC2/ASTC
      - Intel, nVidia (tegra), Mali, Adreno, Apple (Ax and Mx)
    - PVRTC1
  - KTX 2.0 supports Basis Universal compression
    - <https://github.com/BinomialLLC/basis_universal>
    - no direct GPU support
    - can be transcoded to GPU compressed formats quickly
- build
  - `git clone https://github.com/KhronosGroup/KTX-Software.git`
  - `cmake -S. -Bout -GNinja -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=$PWD/out/install`
  - `ninja -C out install`
- meson
  - `dependency('Ktx', method: 'cmake', required: false)`
  - `meson setup out --cmake-prefix-path <path-to-ktx-out-install>`
- cmdline tool
  - `ktx create --format ASTC_4x4_UNORM_BLOCK a.png a.ktx`
    - `--1d` for 1D texture
    - `--cubemap` for cuebmap texture
    - `--depth N` for 3D texture
    - `--layers N` for array texture
    - `--levels N` for mipmap
    - `--generate-mipmap` to generate mipmap
    - `--astc-quality thorough` for quality if format is `ASTC_*`
