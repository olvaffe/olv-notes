3D Techniques
=============

## Lighting

- for each fragment, we want to calculate the light perceived by the camera
- the perceived light is the result of
  - object color
  - ambient light
  - point lights
  - directional lights
  - more
- a simplest model gives
  - `objColor * (ambientColor + directionalColor * dot(objNormal, directionalLightDir))`

## Shadow Mapping

- first pass: render the scene from the light's point of view
  - depth only; no color attachment
  - vs only; no fs
  - a depth value indicates the distance from the light
  - this generates a shadow map
- second pass: render the scene from the camera's point of view
  - vs: calculate positions from the camera's and the light's point of view
  - fs: when the distance from the light is larger than the value stored in
    shadow map, the fragment is not lit by the light
- when there are multiple lights,
  - first pass is done for each light
  - second pass is done once, but all lights are added together in fs

## Deferred Shading

- the straighforward forward shading
  - has complexity of `O(number_of_lights * number_of_objects)`
  - lots of overdrawn when objects occlude each other
- deferred shading
  - has complexity of `O(number_of_lights * window_size)`
  - zero overdrawn
- geometry pass: render the scene to g-buffer
  - vs is the normal vs
  - fs is a special one that writes diffferent infos to different render
    targets
    - one RT for transformed position
    - one RT for transformed normal
    - one RT for object color
    - that is, instead of doing lighting using the infos, we write them out to
      different RTs
  - the collection of the render targets is called g-buffer (geometry buffer)
- lighting pass: render a window-size rectangle
  - vs: do nothing
  - fs: read lighting infos from g-buffer; lighting normally

## Model Rendering

- a 3D model consists of an array of sub-meshes
- each sub-mesh has
  - vertex attributes which are each an array of values
    - position
    - normal
    - texcoord
  - material
    - albedo texture (or constant color)
    - normal texture (or just "normal" vertex attribute)
    - metallic texture (or just a float value)
    - roughness texture (or just a float value)
    - more
  - optionally an array of indices (i.e., index buffer)
- to render a model,
  - switch vbo/ibo/textures/samplers for each sub-mesh
  - vs: transform position and normal
  - fs: sample material textures and shadow maps; apply physically-based
    lighting
- skinning and animation
  - TODO

## Terrain Rendering

- can be rendered like a model
  - highly detailed mesh
  - albedo texture
  - normal texture
- but it is better to do
  - a grid with height map
    - vs samples the height map to adjust position of the grid
  - albedo texture
  - normal texture
- designer usually go like
  - oh, my terrain is going to be 5km x 5km
  - my resolution is 50cm
  - I need 10000 x 10000 grid
- the textures cannot fit GPU memory
- tricks
  - divide terrain into tiles
  - LOD depending on how far a tile is from the camera
    - LOD for both the mesh and the textures
  - dynamic tile streaming and sparse textures
    - alternatively, use a splatmap
