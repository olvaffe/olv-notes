glTF
====

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
