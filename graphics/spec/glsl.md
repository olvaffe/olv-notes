GLSL
====

## Links

- <https://registry.khronos.org/OpenGL/specs/gl/GLSLangSpec.4.60.html>
- <https://github.com/KhronosGroup/GLSL>

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
    - `layout(location = 2, binding = 3) uniform sampler2D s;`
      - this binds the sampler to unit 3
      - same as `glUniformi(2, 3)`
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
  - a matrix is initialized in a column major fashion, just as in
    `glLoadMatrix`.
    - unrelated, but recall that `glMultMatrix` is applied on the right side of
      the current matrix
  - texture-combined sampler constructors
    - a `sampler2D` can be constructed from a separated pair of `sampler s`
      and `texture2D t` through `sampler2D(t, s)`
    - VK only
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
  - matrix multiplication
    - the rule is the same as in math.
    - A left vector operand is treated as a row vector.
    - A right vector operand is treated as a column vector.
    - the number of columns of the left operand must be the same as the number of
      rows of the right operand.
    - the result has the same number of rows as the left operand and the same
      number of columns as the right operand
- 5.11. Out-of-Bounds Accesses
- 5.12. Specialization-Constant Operations

## Chapter 6. Statements and Structure

- 6.1. Function Definitions
  - functions can be overloaded
  - `subroutine`
- 6.2. Selection
  - `if`, `if-else`, and `switch`
- 6.3. Iteration
  - `for`, `while`, and `do-while`
- 6.4. Jumps
  - `continue` and `break`
  - `return`
  - `discard`, only available in FS

## Chapter 7. Built-In Variables

- 7.1. Built-In Language Variables
  - vs inputs
    - gl-only: `gl_VertexID` and `gl_InstanceID`
    - vk-only: `gl_VertexIndex` and `gl_InstanceIndex`
    - `gl_BaseVertex`, `gl_BaseInstance`, and `gl_DrawID`
    - spirv
      - `VertexId`
      - `InstanceId`
      - `VertexIndex`
      - `InstanceIndex`
      - `BaseVertex`
      - `BaseInstance`
      - `DrawIndex`
    - mesa
      - see `vtn_get_builtin_location`
      - `SYSTEM_VALUE_VERTEX_ID`
        - equivalent to `SYSTEM_VALUE_VERTEX_ID_ZERO_BASE + SYSTEM_VALUE_FIRST_VERTEX`
      - `SYSTEM_VALUE_INSTANCE_ID`
      - `SYSTEM_VALUE_VERTEX_ID`
      - `SYSTEM_VALUE_INSTANCE_INDEX`
      - `SYSTEM_VALUE_BASE_VERTEX` (gl) or `SYSTEM_VALUE_FIRST_VERTEX` (vk)
      - `SYSTEM_VALUE_BASE_INSTANCE`
      - `SYSTEM_VALUE_DRAW_ID`
    - with `glDrawArraysInstancedBaseInstance(mode, first, count, instancecount, baseinstance)`
      - `gl_VertexID` is between `[first, first + count)`
      - `gl_InstanceID` is between `[0, instancecount)`
        - note the inconsistency from `gl_VertexID`!
      - `gl_BaseVertex` is 0
      - `gl_BaseInstance` is `baseinstance`
      - `gl_DrawID` is 0
    - with `glDrawElementsBaseVertex(..., indices, basevertex)`
      - `gl_VertexID` is `indices[i] + basevertex`
      - `gl_BaseVertex` is `basevertex`
    - with `glMultiDrawArrays(..., drawcount)`
      - `gl_DrawID` is between `[0, drawcount)`
    - with `vkCmdDraw(cmd, vertexCount, instanceCount, firstVertex, firstInstance)`
      - `gl_VertexIndex` is between `[firstVertex, firstVertex + vertexCount)`
        - same as `gl_VertexId`
      - `gl_InstanceIndex` is between `[firstInstance, firstInstance + instancecount)`
        - different from `gl_InstanceId`
      - `gl_BaseVertex` is `firstVertex`
        - different from gl
      - `gl_BaseInstance` is `firstInstance`
        - same as gl
      - `gl_DrawID` is 0
        - same as gl
    - with `vkCmdDrawIndexed(cmd, indexCount, instanceCount, firstIndex, vertexOffset, firstInstance)`
      - `gl_VertexIndex` is `indices[i] + vertexOffset`
      - `gl_BaseVertex` is `vertexOffset`
    - with `vkCmdDrawMultiEXT(cmd, drawCount, ...)`
      - `gl_DrawID` is between `[0, drawCount)`
  - vs outputs:
    - `gl_PerVertex`
      - `gl_Position`
        - undefined if not written to
      - `gl_PointSize`
        - undefined if not written to
      - `gl_ClipDistance[]`
        - must be written to for the user clip planes that are enabled.  If
          not, the fixed-function may calculate the clip distances to user
          planes from `gl_Position`, but it is deprecated.
      - `gl_CullDistance[]`
  - tcs inputs
    - `gl_PerVertex gl_in[gl_MaxPatchVertices]`
    - `gl_PatchVerticesIn`, `gl_PrimitiveID`, `gl_InvocationID`
  - tcs outputs
    - `gl_PerVertex gl_out[]`
    - `gl_TessLevelOuter[4]` and `gl_TessLevelInner[2]`
  - tes inputs
    - `gl_PerVertex gl_in[gl_MaxPatchVertices]`
    - `gl_PatchVerticesIn` and `gl_PrimitiveID`
    - `gl_TessCoord`
    - `gl_TessLevelOuter[4]`and `gl_TessLevelInner[2]`
  - tes outputs
    - `gl_PerVertex`
  - gs inputs
    - `gl_PerVertex gl_in[]`
    - `gl_PrimitiveIDIn` and `gl_InvocationID`
  - gs outputs
    - `gl_PerVertex`
    - `gl_PrimitiveID`, `gl_Layer`, `gl_ViewportIndex`
  - fs inputs
    - `gl_FragCoord`
    - `gl_FrontFacing`
    - `gl_ClipDistance[]`
    - `gl_CullDistance[]`
    - `gl_PointCoord`
      - defined only for point sprite
    - `gl_PrimitiveID`
    - `gl_SampleID`
    - `gl_SamplePosition`
    - `gl_SampleMaskIn[]`
    - `gl_Layer`
    - `gl_ViewportIndex`
    - `gl_HelperInvocation`
  - fs outputs
    - `gl_FragDepth`
    - `gl_SampleMask[]`
  - compute inputs
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
  - compatibility profile
    - `gl_PerVertex` gains
      - `gl_ClipVertex`
      - `gl_FrontColor`
      - `gl_BackColor`
      - `gl_FrontSecondaryColor`
      - `gl_BackSecondaryColor`
      - `gl_TexCoord[]`
      - `gl_FogFragCoord`
    - fs input: `gl_PerFragment`
      - `gl_FogFragCoord`
      - `gl_TexCoord[]`
      - `gl_Color`
      - `gl_SecondaryColor`
    - fs outputs
      - `gl_FragColor`
      - `gl_FragData`
        - it is deprecated because `glBindFragDataLocation` can bind arbitrary
          output variable to a color number.
- 7.2. Compatibility Profile Vertex Shader Built-In Inputs
  - `gl_Color` and `gl_SecondaryColor`
  - `gl_Normal`
  - `gl_Vertex`
  - `gl_MultiTexCoord0` to `gl_MultiTexCoord7`
  - `gl_FogCoord`
- 7.3. Built-In Constants
  - `gl_Max*`
- 7.4. Built-In Uniform State
  - gl-only
    - `gl_DepthRange`
    - `gl_NumSamples`
  - compatibility profile
    - `gl_*Matrix` such as `gl_ModelViewMatrix`
    - `gl_*MatrixInverse`
    - `gl_NormalScale`
    - `gl_ClipPlane`
    - `gl_Point`
    - `gl_FrontMaterial` and `gl_BackMaterial`
    - `gl_*Light*` such as `gl_LightSource`
    - `gl_TextureEnvColor`
    - `gl_EyePlane{S,T,R,Q}`
    - `gl_ObjectPlane{S,T,R,Q}`
    - `gl_Fog`
- 7.5. Redeclaring Built-In Blocks
  - `out gl_PerVertex` can be redeclared to explicitly indicate the subset of
    members accessed

## Chapter 8. Built-In Functions

- 8.1. Angle and Trigonometry Functions
  - `radians`, `degrees`
  - `sin`, `cos`, `tan`
  - `asin`, `acos`, `atan`
  - `sinh`, `cosh`, `tanh`
  - `asinh`, `acosh`, `atanh`
- 8.2. Exponential Functions
  - `pow`, `exp`, `log`
  - `exp2`, `log2`
  - `sqrt`, `inversesqrt`
- 8.3. Common Functions
  - `abs`, `sign`
  - `floor`, `trunc`, `round`, `roundEven`, `ceil`, `fract`
  - `mod`, `modf`
  - `min`, `max`, `clamp`
  - `mix`, `step`, `smoothstep`
  - `isnan`, `isinf`
  - `floatBitsToInt`, `floatBitsToUint`, `intBitsToFloat`, `uintBitsToFloat`
  - `fma` returns `a * b + c` with a twist
    - if the returned value is eventually consumed by a variable declared as
      `precise`,
      - `a * b + c` is two operations and is rounded twice
      - `fma(a, b, c)` is one operation and is rounded once
    - othterwise, the two are the same
  - `frexp`, `ldexp`
- 8.4. Floating-Point Pack and Unpack Functions
  - `packUnorm2x16`, `packSnorm2x16`, `packUnorm4x8`, `packSnorm4x8`
  - `unpackUnorm2x16`, `unpackSnorm2x16`, `unpackUnorm4x8`, `unpackSnorm4x8`
  - `packHalf2x16`, `unpackHalf2x16`
  - `packDouble2x32`, `unpackDouble2x32`
- 8.5. Geometric Functions
  - `length`, `distance`, `dot`, `cross`, `normalize`
  - compat-only: `ftransform`,
  - `faceforward`
  - `reflect`, `refract`
- 8.6. Matrix Functions
  - `matrixCompMult`, `outerProduct`, `transpose`, `determinant`, `inverse`
- 8.7. Vector Relational Functions
  - `lessThan`, `lessThanEqual`, `greaterThan`, `greaterThanEqual`, `equal`, `notEqual`
  - `any`, `all`
  - `not`
- 8.8. Integer Functions
  - `uaddCarry`, `usubBorrow`, `umulExtended`
  - `bitfieldExtract`, `bitfieldInsert`, `bitfieldReverse`
  - `bitCount`, `findLSB`, `findMSB`
- 8.9. Texture Functions
  - `textureSize` queries the size of the lod
  - `textureQueryLod` queries the lods that would be accessed
  - `textureQueryLevels` queries the available lods
  - `textureSamples` queries the sample count
  - `texture` performs texture lookup
    - it supports all samplers, including shadow and array samplers
  - `textureProj` is the same as `texture`, after projecting the texcoords
  - `textureLod` is the same as `texture`, except with explicit lod
  - `textureOffset` is the same as `texture`, except an offset is applied
    - useful for box blur, and hw might be able to prefetch?
  - `texelFetch` fetches a specific texel
  - `texelFetchOffset` is the same as `texelFetch`, except an offset is
    applied
  - `textureProjOffset`
  - `textureLodOffset`
  - `textureProjLod`
  - `textureProjLodOffset`
  - `textureGrad` is the same as `texture`, except with explicit gradients
  - `textureGradOffset`
  - `textureProjGrad`
  - `textureProjGradOffset`
  - `textureGather` fetches 4 texels that would be fetched and used for linear
    filtering
  - `textureGatherOffset`
  - `textureGatherOffsets`
  - compat-only
    - `texture1D{,Proj,Lod,ProjLod}`
    - `texture2D{,Proj,Lod,ProjLod}`
    - `texture3D{,Proj,Lod,ProjLod}`
    - `textureCube{,Lod}`
    - `shadow1D{,Proj,Lod,ProjLod}`
    - `shadow2D{,Proj,Lod,ProjLod}`
- 8.10. Atomic Counter Functions
  - `atomicCounterIncrement`, `atomicCounterDecrement`, `atomicCounter`
  - `atomicCounterAdd`, `atomicCounterSubtract`
  - `atomicCounterMin`, `atomicCounterMax`
  - `atomicCounterAnd`, `atomicCounterOr`, `atomicCounterXor`
  - `atomicCounterExchange`, `atomicCounterCompSwap`
- 8.11. Atomic Memory Functions
  - `atomicAdd`
  - `atomicMin`, `atomicMax`
  - `atomicAnd`, `atomicOr`, `atomicXor`
  - `atomicExchange`, `atomicCompSwap`
- 8.12. Image Functions
  - `imageSize`, `imageSamples`
  - `imageLoad`, `imageStore`
  - `imageAtomicAdd`
  - `imageAtomicMin`, `imageAtomicMax`
  - `imageAtomicAnd`, `imageAtomicOr`, `imageAtomicXor`
  - `imageAtomicExchange`, `imageAtomicCompSwap`
- 8.13. Geometry Shader Functions
  - `EmitStreamVertex`, `EndStreamPrimitive`
  - `EmitVertex`, `EndPrimitive`
- 8.14. Fragment Processing Functions
  - `dFdx`, `dFdy`, `dFdxFine`, `dFdyFine`, `dFdxCoarse`, `dFdyCoarse`
  - `fwidth`, `fwidthFine`, `fwidthCoarse`
  - `interpolateAtCentroid`, `interpolateAtSample`, `interpolateAtOffset`
- 8.15. Noise Functions
  - deprecated: `noise1` to `noise4`
- 8.16. Shader Invocation Control Functions
  - `barrier`
- 8.17. Shader Memory Control Functions
  - `memoryBarrier`
  - `memoryBarrierAtomicCounter`
  - `memoryBarrierBuffer`
  - `memoryBarrierShared`
  - `memoryBarrierImage`
  - `groupMemoryBarrier`
- 8.18. Subpass-Input Functions
  - `subpassLoad`
- 8.19. Shader Invocation Group Functions
  - `anyInvocation`, `allInvocations`, `allInvocationsEqual`

## Chapter 9. Shading Language Grammar

## Chapter 10. Acknowledgments

## Chapter 11. Normative References

## Chapter 12. Non-Normative SPIR-V Mappings

- 12.1. Feature Comparisons
- 12.2. Mapping from GLSL to SPIR-V

## Extensions

- <https://github.com/KhronosGroup/GLSL/blob/main/extensions/ext/GL_EXT_control_flow_attributes.txt>
  - `[[unroll]]`
  - `[[dont_unroll]]`
  - `[[dependency_infinite]]`
  - `[[dependency_length(N)]]`
  - `[[flatten]]`
  - `[[dont_flatten]]`
- <https://github.com/KhronosGroup/GLSL/blob/main/extensions/ext/GL_EXT_null_initializer.txt>
  - `{}`, such as `vec4 x = {};`
- <https://github.com/KhronosGroup/GLSL/blob/main/extensions/ext/GL_EXT_samplerless_texture_functions.txt>
  - texture functions that do not require a sampler (`texelFetch`,
    `textureSize`, etc.) can be used with `gtexture*D*`
- <https://github.com/KhronosGroup/GLSL/blob/main/extensions/ext/GL_EXT_shader_explicit_arithmetic_types.txt>
  - explicitly-sized ints and floats

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
