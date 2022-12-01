GLSL
====

## GLSL (1.3)

- Matrix
  - `mat2x3` is a 3 by 2 matrix.  That is, 3 rows and 2 columns.
  - a matrix is initialized in a column major fashion, just as in
    `glLoadMatrix`.
    - unrelated, but recall that `glMultMatrix` is applied on the right side of
      the current matrix
  - Multiplications of two matrices or one matrix and one vector is defined in
    `Expressions`
    - A right vector operand is treated as a column vector.  A left vector operand
      is treated as a row vector.
    - the rule is the same as in math.
    - the number of columns of the left operand must be the same as the number of
      rows of the right operand.
    - the result has the same number of rows as the left operand and the same
      number of columns as the right operand
- `#version 130` is required for GLSL after 1.30
  - 1.10 is assumed if not specified
- different shaders linked together must have the same version
- By default, `#extension all : disable` is declared
- all variables and functions must be declared before being used
  - they must have types and optionally qualifiers
- basic types
  - void, bool, int, uint, float
  - vec2, vec3, vec4, and boolean and signed/unsigned integer variants
  - mat2, mat3, mat4, and matMxN (column major)
  - samplers
    - only in function parameters or uniforms
  - struct
  - NO pointers
- Arrays
  - arrays may be declared with or without a size
  - an array without a size can only be indexed with a constant expression
    - so that the compiler can derive its size
- implicit conversions
  - int and uint to float
  - ivec[234] and uvec[234] to vec[234]
- storage qualifiers
  - local read/write memory if no qualifier
  - `const`: a compile-time constant or a function parameter that is read-only
  - `in`, `centroid in`: linkage into a shader from a previous stage
    - `attribute`: deprecated
    - `centroid`: is ignored for single-sample rasterization.  For multisample
      rasterization, the interpolation must be evaluated at any location in the
      pixel or any of the sample point, and the location must fall inside the
      primitive.  When `centroid` is not given, the interpolation may be
      evaluated at any location in the pixel or any of the sample point.
  - `out`, `centroid out`: linkage out of a shader to a subsequent stage
    - `varying`, `centroid varying`: deprecated
  - `uniform`
  - outputs from a VS and inputs to a FS can be further qualified with
    - `smooth`: perspective correct interpolation
    - `flat`: no interpolation
    - `noperspective`: linear interpolation
- It is expected that the graphic hardware will have a small number of fixed
  vector locations in FS.  Each non-matrix input variable takes up one such
  location.  A matrix input takes up multisample locations, equal to the number
  of columns.
- parameter qualifiers
  - `in`, `out`, `inout`
- precision qualifiers
  - `highp`, `mediump`, and `lowp`
  - the syntax is defined, but it has not effect
  - `precision <precision-qualifier> <type>` can change the default precision of
    a type
- invariant qualifiers
  - a output variable may have different values from the same expression in
    different shaders.  To avoid that for multi-pass rendering, `invariant` may
    be used.
- vector components
  - `{x,y,z,w}`
  - `{r,g,b,a}`
  - `{s,t,p,q}`
- functions
  - functions can be overloaded
- Built-in Variables
  - VS special variables
  
      out vec4  gl_Position;       // must be written to
      float     gl_PointSize;      // may be written to
      in  int   gl_VertexID;
      out float gl_ClipDistance[]; // may be written to
      out vec4  gl_ClipVertex;     // may be written to, deprecated
    - `gl_ClipDistance` must be written to for the user clip planes that are
      enabled.  If not, the fixed-function may calculate the clip distances to
      user planes from `gl_Position`, but it is deprecated.
  - FS special variables
  
      in  vec4  gl_FragCoord;
      in  bool  gl_FrontFacing;
      in  float gl_ClipDistance[];
      out vec4  gl_FragColor;                   // deprecated
      out vec4  gl_FragData[gl_MaxDrawBuffers]; // deprecated
      out float gl_FragDepth;
  - VS built-in inputs

      in vec4  gl_Color;          // deprecated
      in vec4  gl_SecondaryColor; // deprecated
      in vec3  gl_Normal;         // deprecated
      in vec4  gl_Vertex;         // deprecated
      in vec4  gl_MultiTexCoord0; // deprecated
      in vec4  gl_MultiTexCoord1; // deprecated
      in vec4  gl_MultiTexCoord2; // deprecated
      in vec4  gl_MultiTexCoord3; // deprecated
      in vec4  gl_MultiTexCoord4; // deprecated
      in vec4  gl_MultiTexCoord5; // deprecated
      in vec4  gl_MultiTexCoord6; // deprecated
      in vec4  gl_MultiTexCoord7; // deprecated
      in float gl_FogCoord;       // deprecated
  - Built-in Uniform State
    - a whole lot of uniforms for current matrices, material, and etc.  All
      deprecated.
  - Built-in VS output variables

      out vec4  gl_FrontColor;          // deprecated
      out vec4  gl_BackColor;           // deprecated
      out vec4  gl_FrontSecondaryColor; // deprecated
      out vec4  gl_BackSecondaryColor;  // deprecated
      out vec4  gl_TexCoord[]; // deprecated, at most will be gl_MaxTextureCoords
      out float gl_FogFragCoord;// deprecated
  - Built-in FS input variables

      in   vec2   gl_PointCoord;
      in   float  gl_FogFragCoord;              //   deprecated
      in   vec4   gl_TexCoord[];                //   deprecated
      in   vec4   gl_Color;                     //   deprecated
      in   vec4   gl_SecondaryColor;            //   deprecated
    - `gl_PointCoord` is defined only for point sprite
- Why is `gl_FragColor` deprecated?
  - `glBindFragDataLocation` can bind arbitrary output variable to a color
    number.
- Why is `gl_Position` optional in GLSL 1.4?
  - because it may be the GS to write it?

## GLSL texture

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

## Chapter 4. Variables and Types

- there are texture-combined sampler types
  - `gsampler*`
  - both GL and VK use them
- there are also (separated) texture and sampler types
  - VK only
- `layout(location=2, binding=3) uniform sampler2D s;`
  - this binds the sampler to unit 3
  - `glUniformi(2, 3)`

## Chapter 5. Operators and Expressions

- texture-combined sampler constructors
  - a `sampler2D` can be constructed from a separated pair of `sampler s` and
    `texture2D t` through `sampler2D(t, s)`
  - VK only

## Chapter 6. Statements and Structure

## Chapter 7. Built-In Variables

## Chapter 8. Built-In Functions

- to sample a texture-compined sampler, `texture()`

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
