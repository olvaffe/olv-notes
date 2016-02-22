llvmpipe
========

## `llvmpipe_resource`

* A tile is 64x64
* A `llvmpipe_resource` has per-level
  * `row_stride` and `img_stride`.  `img_stride` is used for layered
    (`depth > 1`) resource.
  * `tiles_per_row` and `tiles_per_image`
  * `num_slices_faces`.  That is, number of layers.
  * `layout` record to track `lp_texture_layout` of each tile
  * `tiled` and `linear` for tiled and linear data.  Tiled or linear, multiple
    layers are stored in a malloc'ed block.  A layer is called an image in
    llvmpipe.
* As can be seen from `llvmpipe_resource_map`, `lp_texture_layout` is
  * `LP_TEX_LAYOUT_NONE` if neither linear nor tiled data is valid
  * `LP_TEX_LAYOUT_TILED` if only the tiled data is valid
  * `LP_TEX_LAYOUT_LINEAR` if only the linear data is valid
  * `LP_TEX_LAYOUT_BOTH` if both linear and tiled data are valid
  * the format of tiled data is always `PIPE_FORMAT_B8G8R8A8_UNORM`
  * when a `LP_TEX_LAYOUT_LINEAR` tile is mapped as tiled, the tiled data
    storage are allocated, and the linear data are swizzled.  If it is a
    read-write map, the new layout is `LP_TEX_LAYOUT_TILED`.  If it is a
    read-only map, the new layout is `LP_TEX_LAYOUT_BOTH`.
* To implement `resource_copy_region`,
  * the region in both src and dst is set to linear layout.  `util_copy_rect` is
    called to do the copy.
* `clear` is used to clear a `pipe_surface` bound to the framebuffer.
  `clear_render_target` and `clear_depth_stencil` are used to clear a
  `pipe_surface` that may or may not be bound to the framebuffer.

## State Update

* When any of the Draw functions is called, `llvmpipe_update_derived` is called
  first.  It will propogate the state changes into the setup context and clear
  `llvmpipe->dirty`.
  * If any resource is mapped for write, `LP_NEW_SAMPLER_VIEW` is set to notify
    the llvmpipe context. 
  * the vertex info is recalculated.
  * if the fragment pipeline needs to change, a new variant is installed into
    the setup context.
  * Similarly, blend state, dsa state, constants, and new samplers changes are
    propogated into the setup context.
* Just before the setup context starts rasterization, `lp_setup_update_state` is
  called to clear `lp->dirty`, the dirty state of the setup context.
  * the scene is flushed if the size exceeds the limit
  * 

## scene and rast

* When the setup context is initialized, an `lp_rasterizer` and two `lp_scene`s
  are created
* A scene has
  * the `pipe_framebuffer_state`
  * a list of resources
  * the scene size (i.e. currently used bytes of the rast data and resources)
  * A command bin for each tile
    * A bin has a list of command blocks (A command block can hold 128 commands)
  * A list of data block (16KiB each)
* A rasterizer has
  * a scene list to hold scenes that are full
  * a `lp_rasterizer_task` for each thread
  * a barrier
  * a number of threads
* Lifecycle of a scene
  * When the state of the setup context is updated, `lp_setup_get_current_scene`
    is called to install an empty scene as the current scene of the setup
    context, if there is no current scene.
    * `lp_scene_begin_binning` is called to bind the empty scene to the fb state.
      * it copies the fb state into the scene and calculate the number of tiles of
        the fb.
    * `lp_scene_bin_state_command` is called to add `lp_set_state` command to the
      bin of each tile
    * all sampler resources are added to the list of scene resources.
  * the `setup->tri` is called because of vbuf rendering or pipeline rendering
    * the current scene is set to `SETUP_ACTIVE`
    * `begin_binning` is called.  It installs `lp_rast_clear_color` and/or
      `lp_rast_clear_zstencil` if there are clear queued (`glClear` while the
      current scene is not active).
    * Triangle rasterization.  A `lp_rast_triangle` is allocated from the
      scene's data blocks.  A `lp_rast_shade_tile` is added to the bin if a tile
      is fully covered.  A `lp_rast_triangle` is added to the bin if a tile is
      only partially covered.
  * the current scene is flushed if its size exceeds a limit or `glFlush` or a
    new fb is bound.
    * `lp_setup_rasterize_scene` is called.  It queues the current scene into
      the `lp_rasterizer` and signals all rasterizer threads.  The scene is now
      in the hands of the rasterizer threads.
    * the other threads wait for the first thread to finish `lp_rast_begin`.
      `lp_rast_begin` makes sure the tiled data storage is allocated by calling
      `llvmpipe_resource_map` with `LP_TEX_LAYOUT_NONE`.
    * it calls `lp_rast_finish` to wait all rasterizer threads to be done and
      adds the scene back into the empty scene queue.  NO DOUBLE-SCENE RENDERING
      HERE!!

## Rasterize a scene

* after a scene is passed to the rasterizer, multiple rasterizer threads will
  call `rasterize_scene` on the same scene.
* For each bin, it calls `lp_rast_tile_begin` to get the pointers to the color
  buffers and depth buffer.  It calls `llvmpipe_get_texture_tile` to get the
  pointers, storing them in `color_tiles` and `depth_tile`
* Then each command of a bin is executed in order. 

## Rasterization of a triangle

* `generate_setup_variant` creates a JIT function to calculate `a0`, `dadx`,
  `dady` for each attribute from the vertices
  * It is to be noted that barycentric interpolation is linear.  To see why,
    expand the areas using the area formula.  The interpoloated value at
    (x, y) can thus be expressed as `a0 + dadx * x + dady * y`
  * Let `v0 = (x0, y0, z0), v1 = (x1, y1, z1), v2 = (x2, y2, z2), where z's
    stand for attribute values at v's`.  Then these vertices are on the plane
    `Ax + By + Cz = D`.  Or, `z = A'x + B'y + C'`, which is exactly `a0`,
    `dadx`, and `dady`.
    * `(A, B, C)` is `cross(v0v1, v0v2)`
  * `LP_INTERP_CONSTANT` emits the attr of the provoking vertex 
  * `LP_INTERP_FACING` emits `(bool, 0, 0, 0)` where bool is true when the
    primitive is front facing
  * `LP_INTERP_PERSPECTIVE` calculates perspective-correct `a0, dadx, and dady`
    Later, we will divide the interpolated value by interpolated w when the
    shader is run
  * because OpenGL samples at the center of pixels, while gallium samples at
    the top-left corner of pixels, `a0` is calculated as if the primitive is
    translated by `(-0.5, -0.5)`.
* `calc_fixed_position`
  * snap vertices to subpixel grid
    * in other words, convert floats to fixed
  * get dx and dy
    * in gallium, Y-axis goes downward and the sign of dy is inverted.
      As such, one of the vector is from V1 to V0, not from V0 to V1.
  * calculate the area
    * twice the area actually
* `do_triangle_ccw` queues bin commands to the intersected tiles
* `lp_rast_triangle`
  * <http://devmaster.net/forums/topic/1145-advanced-rasterization/>
  * Given v0 = (x0, y0), v1 = (x1, y1), the half-space function is
    `E(x, y) = vx - uy - (vx0 - uv0), where u = x1 - x0, v = y1 - y0`
    * the function has the property that
      * `E(x, y) = 0` for (x, y) on the line
      * `E(x, y) > 0` for (x, y) on the right-hand side
      * `E(x, y) < 0` for (x, y) on the left-hand side
      * It can be easily proved by using (u, v) and (v, -u) as the basis
    * And E(x + 1, y) = E(x, y) + v, E(x, y + 1) = E(x, y) - u.  This allows
      fast computation!
  * with a coordinate system that y-axis points downward, the idea of left and
    right are reversed: the half-space function is greater than 0 for (x, y)
    on the left-hand side
    * given this, when v0, v1, v2 are oriented counter-clockwise, points
      lie inside the triangle if and only if all three half-space functions
      evalute to positive values.
  * however, there is floating-point accuracy issue.  The solution is to use
    28.4 fixed-point numbers
    * geometrically, it subdivides each pixel into 16x16 subpixel grid.
      Points are snapped to subpixel grid.
  * there is also issue with points on shared edges.  We can employ top-left
    fill convention to resolve it.
    * a point is regarded as inside a triangle if it is on the top-left edges
      of the triangle.  Since the vertices are oriented counter-clockwise, an
      edge is top-left if it points downward or left.
    * GL does not define a fill convention.  i965 uses top-left.  llvmpipe,
      however, uses bottom-left.
  * With fixed-point, the half-space functions evalute to integers.  We can
    add 1 to the top-left ones so that `E(x, y) > 0` means a point is either
    on the edge or lies on the left-hand side.
  * half-space or edge functions are named planes in the code
* `lp_setup_bin_triangle`
  * `dx` is such that x0, x1, y0, y1 divided by `dx<<1` is the same
    * it allows us to know if a triangle is entirely in a tile
    * `x0 ^ x1` gives the differences between x0 and x1.  `dx` is such that
      `(x0 ^ x1) >> dx` is 1
  * if the triangle is inside a tile, emit one of
    * `LP_RAST_OP_TRIANGLE_3_4`
    * `LP_RAST_OP_TRIANGLE_3_16`
    * `LP_RAST_OP_TRIANGLE_x`, where x is the number of planes (3 or 7)
      * The argument gives the planes that cut the tile.  In this case, all
        planes do.
  * A tile is entirely outside a triangle if all corners are outside the
    triangle.  But we can also check if
    * the top-left corner is to the right of a right-bottom edge, or
    * the bottom-left corner is to the right of a right-top edge, or
    * the bottom-right corner is to the right of a top-left edge, or
    * the top-right corner is to the right of a bottom-left edge
    * That is, given an edge, we know which corner should we test for.
      `plane.eo` can be added to the value at the top-left corner to get the
      value at the desired corner.
  * A tile is partially covered by a triangle if it is not
    entirely outside the triangle and
    * the top-left corner is to the right of a top-left edge, or
    * the bottom-left corner is to the right of a bottom-left edge, or
    * the bottom-right corner is to the right of a bottom-right edge, or
    * the top-right corner is to the right of a top-right edge, or
    * `plane.eo` can be subtracted from the vaule at the bottom-right corner
      to get the vlaue at the desired corner.
  * In the code, `out` is non-zero when the tile is entirely outside the
    triangle.  `partial` is non-zero when the tile is partially in the
    triangle, and its value gives a edge mask giving the edges that intersect
    the tile.
* `lp_rast_triangle_x`
  * it breaks tiles into 16x16 blocks, then blocks into 4x4 subblocks.

## Fragment Pipeline

* The pipeline references `jit_context`.
* The pipeline is codegened when `llvmpipe_update_fs` is called.  The pipeline
  is encapsulated inside a variant.
* A variant is identified by a `lp_fragment_shader_variant_key`
  * a key copies all relevant states from the llvmpipe context into the key
* The pipeline processes a 2x2 quads at a time.  A quads has 2x2 pixels.
  * the pipeline processes quad 0, 1, 2, 3 in order.  For each quad, 4 pixels
    are processed as `<4 x float>`
  * first, it interpolates the attributes from a0, dadx, and dady arrays using
    the SOA context.
  * it then builds scissor test code.
  * it calls `lp_build_tgsi_soa` to build fragment shader.  The outputs are
    stored in the stack using `alloca`.
  * after shader, the outputs (colors and depth) are remembered.  Depth is used
    for depth test.  A color mask of type `4 x <4 x i32>` is also returned.
  * finally, the colors (`4 x <4 x float>`) are converted to `<16 x i8>` for
    blending.  The sequence is load, blend, and store.
* Why does SoA preferred?
  * grep for `LLVMBuildLoad` and `LLVMBuildStore` to see when does the texture
    begin accessed.
  * interp context loads `a0`, `dadx`, and `dady`.  They have nothing to do with
    SoA.
  * in depth test, the depth buffer is loaded.  The type of `depth_ptr`, as can
    be seen in `generate_depth_stencil`, depends on the depth buffer pipe
    format.  The width is equal to the format width.  The length is
    `128 / width` (128bits = 1 quad).  Therefore, a load read a quad (128bits).
    It also needs to store it back.  SoA depth buffer allows us to process a
    quad at a time here.
  * In `generate_blend`, the color buffers are loaded for blending.  SoA color
    buffer also allows us to process a quad at a time.
* Why does tiling preferred?
  * because sampling needs filtering?
  * but in `lp_setup_set_fragment_sampler_views` and in
    `lp_build_sample_offset`, textures use linear layout!

## JIT function and Interpolation

* The JIT function processes 2x2 quads at a time.  That is, 4x4 pixels.
* The inputs to a JIT function are
  * `struct lp_jit_context *context`: the JIT context
  * `uint32_t x, y`: the coordinates from top-left of the render target
  * `float facing`: `> 0` for front-facing, `< 0` for back-facing
  * `const void *a0, *dadx, *dady`: each points to an array of size
                                    `NUM_CHANNELS * (nr_inputs + 1) * sizeof(float)`
                                    `+1` is for the position.
  * `uint8_t **color`: an array of pointers to the color buffers of the render
                       targets at the `(x, y)` coordinates
  * `void *depth`: a pointer to the depth/stencil buffer at the `(x, y)` coordinates
  * `int32_t c1, c2, c3`: `INT_MAX` if no edge test.
  * `int32_t *step1, *step2, *step3`: edge/step info for 3 edges.  Each points
     to 4x4 array.  `NULL` if no edge test.
  * `uint32_t *counter`: occlude counter for visible pixels
* The SOA context is initialized with `lp_build_interp_soa_init`
  * the a's for the 2x2 quads are
    `[a0, a0 + 2 * dadx, a0 + 2 * dady, a0 + 2 * (dadx + dady)]`
  * dadq is defined to be `[0, dadx, dady, dadx + dady]`
  * to process quad `i`, `lp_build_interp_soa_update` is called which
    * `shufflevector %a, undef, <i, i, i, i>`
    * `add %a, %dadq`
    * the result is the attr value sampled at each pixel of the quad.  It is
      stored in `bld->attribs`.  For convenience, `bld->pos` points to
      `bld->attribs[0]`, and `bld->inputs` points to `bld->attribs[1]`.

## Control Flows

* `IF` flow
  * the values of the declared variables will depend on the flow
  * `lp_build_flow_create`
  * `lp_build_flow_scope_begin`
  * `lp_build_flow_scope_declare`: declare a variable used by phi
  * `lp_build_if`
  * if-block
  * `lp_build_else`
  * else-block
  * `lp_build_endif`
  * `lp_build_flow_scope_end`
  * `lp_build_flow_destroy`
* `MASK` flow
  * skip code-block if the mask is 0 (all pixels dead)
  * `lp_build_flow_create`
  * `lp_build_mask_begin`
  * code-block
  * `lp_build_mask_end`
  * `lp_build_flow_destroy`

## Blend

* `4 x <4 x float>` is converted to `<16 x i8>` for blending
  * the floats are clamped if necessary.
  * the floats are converted to unsigned norm `[0, 2^{width} - 1]`
  * finally, `lp_build_resize` is called to convert source to the dest by packing.
* if a channel is not in the colormask, `result = dst`
* else if logicop, `xxx`
* else if blend, `result = term_src OP term_dst = src * factor_src OP dst * factor_dst`
* else, `result = src`

## Sampling

* IT SAMPLES A QUAD AT A TIME.  ALL X, Y, Z, OFFSET, AND ETC. ARE VECTORS.
* The call sequence is `llvmpipe -> gallivm -> llvmpipe -> gallivm`, and finally
  `lp_build_sample_soa` is called.
* If maxfilter == minfilter and mipfilter == none, lod is not needed.
  Otherwise, `lp_build_lod_selector` is called to calculate the lod.
* If no mipfilter, ilevel0 is 0.  If nearest mipfilter, ilevel0 is
  `iround(lod)`.  Otherwise (linear), ilevel0 is `ifloor(lod)` and ilevel1 is
  `ilevel0 + 1`, and `lod_frag`, the fraction of lod is saved.
* the width, height, stride of ileve0 and ilevel1 are calculated.  The data
  pointers to the texels are also saved.
* Finally, `lp_build_sample_mipmap` is called with proper filters and levels.
  * `lp_build_sample_image_nearest` if `PIPE_TEX_FILTER_NEAREST`
  * `lp_build_sample_image_linear` if `PIPE_TEX_FILTER_LINEAR`
  * if mipfilter is linear, the sample function is called twice on ilevel0 and
    ilevel1 and the results are interpolated.
* For nearest sampling, `lp_build_sample_image_nearest` is called
  * it calls `lp_build_sample_wrap_nearest` on `(s, t, r)`
  * it calls `lp_build_sample_texel_soa` to sample the texcoord.
* For linear sampling, `lp_build_sample_image_linear` is called
  * it calls `lp_build_sample_wrap_linear` on `(s, t, r)`
  * it calls `lp_build_sample_texel_soa`  to sample the neighbor pixels of the
    texcoord and interpolate the results.
* In `lp_build_sample_texel_soa`,
  * the texcoord is examined first to see if the border is used
  * `lp_build_fetch_rgba_soa` is called to fetch the texel
  * `apply_sampler_swizzle` is called to swizzle the texel
* In the simplest form of texel fetching in `lp_build_fetch_rgba_soa`,
  * `lp_build_gather`: In the most common `length=4, src_width=32, dst_width=32`
    case, 4 texels (128bytes) will be fetched.
  * `lp_build_unpack_rgba_soa`

## Swizzling

* `lp_tile_swizzle_4ub` swizzles a tile
  * `dst` points to the beginning of the tile in the destination
  * `src` points to the beginning of the source (not the tile!)
  * `src_stride` is the stride of the source
  * `x` and `y` is the position of the tile, in pixels (i.e. always aligned to
    tile size)
  * `tile_pixel_offset(x, y, c)`
