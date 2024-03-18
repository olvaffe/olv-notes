GLSL
====

## Chapter 1. Introduction

- 1.1. Changes
- 1.2. Overview
- 1.3. Error Handling
- 1.4. Typographical Conventions
- 1.5. Deprecation

## Chapter 2. Overview of Shading

- 2.1. Vertex Processor
- 2.2. Tessellation Control Processor
- 2.3. Tessellation Evaluation Processor
- 2.4. Geometry Processor
- 2.5. Fragment Processor
- 2.6. Compute Processor

## Chapter 3. Basics

- 3.1. Character Set and Phases of Compilation
- 3.2. Source Strings
- 3.3. Preprocessor
  - `#version number profile_opt`
    - `#version 100` targets ES
    - `#version 110` is the default when missing
    - `#version 150` or later supports the optional `profile_opt`
      - valid values are `core`, `compatibility`, or `es`
      - when missing, `core` is assumed
    - `#version 300 es` and `#version 310 es` target ES and `profile_opt` is
      required
    - `#version` must occur before anything else except for comments and white
      spaces
  - `#extension extension_name : behavior`
    - `extension_name` is the name of the extension or `all` to indicate all
      supported extensions
    - `behavior` can be `require`, `enable`, `warn`, or `disable`
    - the initial state is `#extension all : disable`
- 3.4. Comments
  - both `//` and `/* */` are supported
- 3.5. Tokens
- 3.6. Keywords
- 3.7. Identifiers
- 3.8. Definitions
  - static use
  - dynamically uniform
  - uniform control flow

## Chapter 4. Variables and Types

- all variables and functions must be declared before being used
  - they must have types and optionally qualifiers
- 4.1. Basic Types
  - basic types are typed defined by keywords
    - Transparent Types
      - `void`, `bool`, `int`, `uint`, `float`, `double`
      - `vec2`, `vec3`, `vec4`, and their boolean/signed/unsigned/double
        variants
      - `mat2`, `mat3`, `mat4`, `matMxN` (column major), and their double
        variants
        - `mat2x3` is a 3 by 2 matrix.  That is, 3 rows and 2 columns.
    - Floating-Point Opaque Types
      - `{sampler,texture,image}1D{,Shadow,Array,ArrayShadow}`
      - `{sampler,texture,image}2D{,Shadow,Array,ArrayShadow,MS,MSArray,Rect,RectShadow}`
      - `{sampler,texture,image}3D`
      - `{sampler,texture,image}Cube{,Shadow,Array,ArrayShadow}`
      - `{sampler,texture,image}Buffer`
      - `subpassInput{,MS}`
    - Signed/unsigned Integer Opaque Types
      - `{i,u}{sampler,texture,image}1D{,Array}`
      - `{i,u}{sampler,texture,image}2D{,Array,MS,MSArray,Rect}`
      - `{i,u}{sampler,texture,image}3D`
      - `{i,u}{sampler,texture,image}Cube{,Array}`
      - `{i,u}{sampler,texture,image}Buffer`
      - `{i,u}subpassInput{,MS}`
      - `atomic_uint`
    - Sampler Opaque Types
      - `sampler{,Shadow}`
    - there is no pointer type
  - composites
    - aggregates, matrices, and vertors are collectively referred to as
      composites
    - an aggregate is a structure or an array
  - opaque types
    - they are accessed via built-in functions, no direct access
    - they can only be declared as function parameters or `uniform`-qualified
      variables
    - there can be both storage and memory qualifiers
      - the storage qualifier qualifies the opaque handle
      - the memory qualifier qualifies the object the opaque handle refers to
    - texture-combined samplers: `sampler2D`, etc.
    - images: `image2D`, etc.
    - atomic counters: `atomic_uint`
      - gl-only
    - texture and sampler types: `texture2D`, `sampler`, etc.
      - vk-only
    - subpass inputs: `subpassInput`, etc.
      - vk-only
  - structures
  - arrays
    - arrays may be declared with or without a size
    - an array without a size can only be indexed with a constant expression
      - so that the compiler can derive its size
    - they support `length()` method
  - implicit conversions
    - `int` can be implicitly converted to `uint`
    - `int`/`uint` can be implicitly converted to `float`
    - `int`/`uint`/`float` can be implicitly converted to `double`
    - the same applies to vectors and matrices
- 4.2. Scoping
- 4.3. Storage Qualifiers
  - there can be at most one storage qualifier
    - no qualifier: local read/write memory, or input function parameter
    - `const`: a compile-time constant or a function parameter that is read-only
    - `in`: linkage into a shader from a previous stage
      - `attribute`: deprecated
    - `out`: linkage out of a shader to a subsequent stage
      - `varying`: deprecated
    - `uniform`: linkage between a shader, API, and the application
    - `buffer`: value is stored in a buffer object
    - `shared`: variable storage is shared across all work items in a
      workgroup
      - CS-only
  - there can be at most one auxiliary storage qualifier for `in`/`out`
    - `centroid`: centroid-based interpolation
      - For multisample rasterization, the interpolation must be evaluated at
        any location in the pixel or any of the sample point, and the location
        must fall inside the primitive.  When `centroid` is not given, the
        interpolation may be evaluated at any location in the pixel or any of
        the sample point.
    - `sample`: per-sample interpolation
    - `patch`: per-tessellation-patch attributes
      - only for TCS `out` and TES `in`
      - Per-patch input variables in the TES are filled with the values of
        per-patch output variables written by the TCS
- 4.4. Layout Qualifiers
  - input
    - `location =`
    - `component =`
    - TES only
      - primitive mode: `triangles`, `quads`, or `isolines`
      - vertex spacing: `equal_spacing`, `fractional_even_spacing`, or
        `fractional_odd_spacing`
      - ordering: `cw` or `ccw`
      - piont mode: `point_mode`
    - GS only
      - primitive: `points`, `lines`, `lines_adjacency`, `triangles`, or
        `triangles_adjacency`
      - invocation count: `invocations =`
    - FS only
      - `gl_FragCoord`: `origin_upper_left` or `pixel_center_integer`
      - there is also `layout(early_fragment_tests) in;`
    - CS only
      - `local_size_x =`
      - `local_size_y =`
      - `local_size_z =`
  - output
    - `location =`
    - `component =`
    - transform feedback
      - `xfb_buffer =`
      - `xfb_offset =`
      - `xfb_stride =`
    - TCS only
      - `vertices =`
    - GS only
      - `points`, `line_strip`, `triangle_strip`
      - `max_vertices =`
      - `stream =`
    - FS only
      - `gl_FragDepth`: `depth_any`, `depth_greater`, `depth_less`, or
        `depth_unchanged`
      - `index =`
  - uniform
    - `location =`
    - vk only
      - `push_constant`
  - subroutine function
    - `index =`
  - uniform and shader storage block
    - `shared`, `packed`, `std140`, `std430`
    - `row_major`, `column_major`
    - `binding =`
    - `offset =`
    - `align =`
  - opaque uniform
    - `binding =`
  - atomic counter
    - `binding =`
    - `offset =`
  - format (for `image2D`, etc.)
    - `rgba32f`, etc.
  - subpass input
    - `input_attachment_index =`
- 4.5. Interpolation Qualifiers
  - `smooth`: perspective correct interpolation
  - `flat`: no interpolation
  - `noperspective`: linear interpolation
- 4.6. Parameter Qualifiers
  - none: same as `in`
  - `const`: const input parameter
  - `in`: input parameter
  - `out`: output parameter
  - `inout`: in/out parameter
- 4.7. Precision and Precision Qualifiers
  - `highp`, `mediump`, and `lowp`
  - the syntax is defined, but it has not effect
  - `precision <precision-qualifier> <type>` can change the default precision of
    a type
- 4.8. Variance and the Invariant Qualifier
  - a output variable may have different values from the same expression in
    different shaders.  To avoid that for multi-pass rendering, `invariant` may
    be used.
- 4.9. The Precise Qualifier
  - `precise`
- 4.10. Memory Qualifiers
  - `coherent`
  - `volatile`
  - `restrict`
  - `readonly`
  - `writeonly`
- 4.11. Specialization-Constant Qualifier
  - `constant_id =`
  - `local_size_x_id =`
  - `local_size_y_id =`
  - `local_size_z_id =`
- 4.12. Order and Repetition of Qualification
- 4.13. Empty Declarations

## Chapter 5. Operators and Expressions

- 5.1. Operators
- 5.2. Array Operations
- 5.3. Function Calls
- 5.4. Constructors
  - texture-combined sampler constructors
    - a `sampler2D` can be constructed from a separated pair of `sampler s`
      and `texture2D t` through `sampler2D(t, s)`
    - VK only
  - a matrix is initialized in a column major fashion, just as in
    `glLoadMatrix`.
    - unrelated, but recall that `glMultMatrix` is applied on the right side of
      the current matrix
  - `layout(location=2, binding=3) uniform sampler2D s;`
    - this binds the sampler to unit 3
    - `glUniformi(2, 3)`
- 5.5. Vector and Scalar Components and Length
  - vector components
    - `{x,y,z,w}`
    - `{r,g,b,a}`
    - `{s,t,p,q}`
- 5.6. Matrix Components
- 5.7. Structure and Array Operations
- 5.8. Assignments
- 5.9. Expressions
- 5.10. Vector and Matrix Operations
  - Multiplications of two matrices or one matrix and one vector is defined in
    `Expressions`
    - A right vector operand is treated as a column vector.  A left vector operand
      is treated as a row vector.
    - the rule is the same as in math.
    - the number of columns of the left operand must be the same as the number of
      rows of the right operand.
    - the result has the same number of rows as the left operand and the same
      number of columns as the right operand
- 5.11. Out-of-Bounds Accesses
- 5.12. Specialization-Constant Operations

## Chapter 6. Statements and Structure

- 6.1. Function Definitions
  - functions can be overloaded
- 6.2. Selection
- 6.3. Iteration
- 6.4. Jumps

## Chapter 7. Built-In Variables

- 7.1. Built-In Language Variables
  - vs
    - `gl_ClipDistance` must be written to for the user clip planes that are
      enabled.  If not, the fixed-function may calculate the clip distances to
      user planes from `gl_Position`, but it is deprecated.
    - Why is `gl_Position` optional in GLSL 1.4?
      - because it may be the GS to write it?
  - fs
    - `gl_PointCoord` is defined only for point sprite
    - Why is `gl_FragColor` deprecated?
      - `glBindFragDataLocation` can bind arbitrary output variable to a color
        number.
  - compute
    - `in uvec3 gl_NumWorkGroups;`
      - the numbers of work groups in each dimension, as specified by
        `vkCmdDispatch`
    - `const uvec3 gl_WorkGroupSize;`
      - the sizes of a work group in each dimension, as specified by
        `local_size_[xyz]`
    - `in uvec3 gl_WorkGroupID;`
      - the work group of the current invocation
      - range is `[0, gl_NumWorkGroups)` for each dimension
    - `in uvec3 gl_LocalInvocationID;`
      - the work item in the work group of the current invocation
      - range is `[0, gl_WorkGroupSize)` for each dimension
    - `in uvec3 gl_GlobalInvocationID;`
      - this is `gl_WorkGroupID * gl_WorkGroupSize + gl_LocalInvocationID` which
        uniquely identify the work item of the current invocation
    - `in uint gl_LocalInvocationIndex;`
      - this is `gl_LocalInvocationID.z * gl_WorkGroupSize.x * gl_WorkGroupSize.y +
        gl_LocalInvocationID.y * gl_WorkGroupSize.x + gl_LocalInvocationID.x`
        which is the 1D representation of `gl_LocalInvocationID`
- 7.2. Compatibility Profile Vertex Shader Built-In Inputs
- 7.3. Built-In Constants
- 7.4. Built-In Uniform State
- 7.5. Redeclaring Built-In Blocks

## Chapter 8. Built-In Functions

- 8.1. Angle and Trigonometry Functions
- 8.2. Exponential Functions
- 8.3. Common Functions
- 8.4. Floating-Point Pack and Unpack Functions
- 8.5. Geometric Functions
- 8.6. Matrix Functions
- 8.7. Vector Relational Functions
- 8.8. Integer Functions
- 8.9. Texture Functions
  - to sample a texture-compined sampler, `texture()`
  - `int/ivec* textureSize(gsampler* sampler[, int lod])`
    - return the dimension of the texture
  - `float/gvec4 texture(gsampler *sampler, float/vec* P[, float bias])`
    - texture lookup
  - `float/gvec4 textureProj(gsampler *sampler, float/vec* P[, float bias])`
    - project `P` and then `texture`
  - `float/gvec4 textureLod(gsampler *sampler, float/vec* P, float lod)`
    - texture lookup with explicit lod
  - `float/gvec4 textureOffset(gsampler *sampler, float/vec* P, int/ivec* offset[, float bias])`
    - texture lookup with offset (to the texture coordinates)
  - `textureProjOffset`: combination of `textureProj` and `textureOffset`
  - `textureLodOffset`: combination of `textureLod` and `textureOffset`
  - `textureProjLod`: combination of `textureProj` and `textureLod`
  - `textureProjLodOffset`: combination of `textureProj`, `textureLod`, and `textureOffset`
  - `textureGrad`
    - texture lookup with explicit gradient
  - `textureGradOffset`, `textureProjGrad`, `textureProjGradOffset`: blah
  - `gvec4 texelFetch(gsampler *sampler, int/ivec P, int lod/sample)`
    - fetch a texel
  - `texelFetchOffset`: `texelFetch` with offset
  - `texture1D*`, `texture2D*`, `texture3D*`, `textureCube*`, `shadow*`
    - deprecated
- 8.10. Atomic Counter Functions
- 8.11. Atomic Memory Functions
- 8.12. Image Functions
- 8.13. Geometry Shader Functions
- 8.14. Fragment Processing Functions
- 8.15. Noise Functions
- 8.16. Shader Invocation Control Functions
- 8.17. Shader Memory Control Functions
- 8.18. Subpass-Input Functions
- 8.19. Shader Invocation Group Functions

## Chapter 9. Shading Language Grammar

## Chapter 10. Acknowledgments

## Chapter 11. Normative References

## Chapter 12. Non-Normative SPIR-V Mappings

- 12.1. Feature Comparisons
- 12.2. Mapping from GLSL to SPIR-V

## C Terminologies

- `http://msdn.microsoft.com/zh-tw/library/ks1txk8c.aspx`
  - `declaration:`
    - `declaration-specifiers init-declarator-list_opt ;`
  - `declaration-specifiers:`
    - `storage-class-specifier declaration-specifiers_opt`
    - `type-specifier declaration-specifiers_opt`
    - `type-qualifier declaration-specifiers_opt`
  - `init-declarator-list:`
    - `init-declarator`
    - `init-declarator-list , init-declarator`
  - `init-declarator:`
    - `declarator`
    - `declarator = initializer`
  - For example, `static const int *fp;`
    - `declaration-specifiers`: `static const int`
      - `storage-class-specifier`: `static`
        - the other storage class specifiers are `extern`, `register`, `typedef`
      - `type-qualifier`: `const`
        - the other type qualifier is `volatile`
      - `type-specifier`: `int`
    - `init-declarator-list`: `*fp`
      - one declarator with no initializer in this example
      - the declarator is modified by `*` to declare a pointer
      - it may also be modified by `[]` or `()` to declare an array or a
        function
  - `declarator:`
    - `pointer_opt direct-declarator`
  - `direct-declarator:`
    - `identifier`
    - `( declarator )`
    - `direct-declarator [ const-expression_opt ]`
    - `direct-declarator ( parameter-list-type )`
  - `struct-specifier` is a `type-specifier`
    - `struct identifier_opt { struct-declaration-list }`
    - `struct identifier`
  - `struct-declaration-list:`
    - `struct-declaration`
    - `struct-declaration struct-declaration-list`
  - `struct-declaration:`
    - `specifier-qualifier-list struct-declarator-list ;`
- A declaration must have at least one declarator, or its type specifier must
  declare a `struct`.
