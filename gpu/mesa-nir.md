# NIR

## GLSL to NIR

 - tools
   - `glsl_compiler` to compile GLSL to GLSL IR
   - `spirv2nir` to compile SPIR-V to NIR
   - `MESA_GLSL=dump` to `_mesa_print_ir` GLSL IR and `nir_print_shader` NIR
 - types
   - `struct gl_shader`, created by glCreateShader
   - `struct gl_shader_program`, created by `glCreateProgram`
   - `struct gl_linked_shader`, inside `struct gl_shader_program` and created
     by `glLinkProgram`
   - `struct gl_program`, inside `struct gl_linked_shader`
 - each GLSL snippet is compiled to GLSL IR using `_mesa_glsl_compile_shader`
   - `_mesa_glsl_parse` is called to parse GLSL into AST
   - `_mesa_ast_to_hir` is called to parse AST into HIR, with validations
   - fundamental passes are applied to gather info and lower HIR to LIR (both
     are GLSL IR)
   - all artifacts are saved to `struct gl_shader`
 - `struct glsl_shader` are collected into a `struct gl_shader_program` and
   linked using `_mesa_glsl_link_shader`
   - it calls common `link_shaders` to cross validate and optimize GLSL IRs
   - it calls backend's `LinkShader` function
     - in st case, `st_link_shader`
     - `st_link_shader` applies more lowering and optimization passes
     - it then calls `st_link_nir` to `st_glsl_to_nir` GLSL IR to NIR for each
       `struct gl_program`
 - at draw time, when all states are known, a variant of a `struct gl_program`
   is created and cached
   - in st case, it first clones NIR, calls `st_finalize_nir`, and call into
     the gallium driver to get the CSO.  It then binds the CSO and call
     `draw_vbo`
   - gallium drivers only see the finalized NIR
 - example
   - `glCreateShader`:
         varying vec4 vFillColor;
         void main() {
           gl_FragColor = vFillColor;
         }
   - `glCompileShader`:
         (
         (declare (location=2 shader_out ) vec4 gl_FragColor)
         (declare (location=25 shader_in ) vec2 gl_PointCoord)
         (declare (location=24 shader_in ) bool gl_FrontFacing)
         (declare (location=0 shader_in ) vec4 gl_FragCoord)
         (declare (location=2 shader_in ) vec4 gl_SecondaryColor)
         (declare (location=1 shader_in ) vec4 gl_Color)
         (declare (location=3 shader_in ) float gl_FogFragCoord)
         (declare (location=4 shader_in ) (array vec4 0) gl_TexCoord)
         (declare (uniform ) (array mat4 8) gl_TextureMatrixInverseTranspose)
         (declare (uniform ) (array mat4 8) gl_TextureMatrixTranspose)
         (declare (uniform ) mat4 gl_ModelViewProjectionMatrixInverseTranspose)
         (declare (uniform ) mat4 gl_ProjectionMatrixInverseTranspose)
         (declare (uniform ) mat4 gl_ModelViewMatrixInverseTranspose)
         (declare (uniform ) mat4 gl_ModelViewProjectionMatrixTranspose)
         (declare (uniform ) mat4 gl_ProjectionMatrixTranspose)
         (declare (uniform ) mat4 gl_ModelViewMatrixTranspose)
         (declare (uniform ) mat4 gl_ModelViewProjectionMatrix)
         (declare (shader_in ) vec4 vFillColor)
         ( function main
          (signature void (parameters)
            (
              (assign  (xyzw) (var_ref gl_FragColor)  (var_ref vFillColor) )
            ))
         )
         )
     - it implicitly declares of built-in variables and uniforms
       - many are outdated and are not used by modern shaders
     - `vFillColor` has location -1 because no explicit location is given
   - `glLinkProgram`:
         (
         (declare (location=31 shader_in ) vec4 vFillColor)
         (declare (location=2 shader_out ) vec4 gl_FragColor)
         (declare (temporary ) vec4 gl_FragColor)
         ( function main
           (signature void (parameters)
             (
               (assign  (xyzw) (var_ref gl_FragColor)  (var_ref vFillColor) )
               (assign  (xyzw) (var_ref gl_FragColor@4)  (var_ref gl_FragColor) )
             ))
         )
         )
     - dead variables are eliminated
     - locations are assigned to `vFillColor`
     - NIR is also created
           shader: MESA_SHADER_FRAGMENT
           name: GLSL1
           inputs: 0
           outputs: 0
           uniforms: 0
           shared: 0
           decl_var shader_in INTERP_MODE_NONE float vFillColor (VARYING_SLOT_VAR0.x, 0, 0)
           decl_var shader_in INTERP_MODE_NONE float vFillColor@0 (VARYING_SLOT_VAR0.y, 0, 0)
           decl_var shader_in INTERP_MODE_NONE float vFillColor@1 (VARYING_SLOT_VAR0.z, 0, 0)
           decl_var shader_in INTERP_MODE_NONE float vFillColor@2 (VARYING_SLOT_VAR0.w, 0, 0)
           decl_var shader_out INTERP_MODE_NONE vec4 gl_FragColor (FRAG_RESULT_COLOR, 0, 0)
           decl_function main (0 params)
           
           impl main {
                   decl_var  INTERP_MODE_NONE vec4 out@gl_FragColor-temp
                   decl_var  INTERP_MODE_NONE vec4 in@vFillColor-temp
                   decl_var  INTERP_MODE_NONE vec4 gl_FragColor@3
                   block block_0:
                   /* preds: */
                   vec1 32 ssa_48 = deref_var &vFillColor (shader_in float) 
                   vec1 32 ssa_49 = intrinsic load_deref (ssa_48) (0) /* access=0 */
                   vec1 32 ssa_50 = deref_var &vFillColor@0 (shader_in float) 
                   vec1 32 ssa_51 = intrinsic load_deref (ssa_50) (0) /* access=0 */
                   vec1 32 ssa_52 = deref_var &vFillColor@1 (shader_in float) 
                   vec1 32 ssa_53 = intrinsic load_deref (ssa_52) (0) /* access=0 */
                   vec1 32 ssa_54 = deref_var &vFillColor@2 (shader_in float) 
                   vec1 32 ssa_55 = intrinsic load_deref (ssa_54) (0) /* access=0 */
                   vec4 32 ssa_56 = vec4 ssa_49, ssa_51, ssa_53, ssa_55
                   vec1 32 ssa_6 = deref_var &gl_FragColor (shader_out vec4) 
                   intrinsic store_deref (ssa_6, ssa_56) (15, 0) /* wrmask=xyzw */ /* access=0 */
                   /* succs: block_1 */
                   block block_1:
           }

## SPIR-V to NIR

 - `spirv_to_nir`

## NIR to backend

 - after translation from GLSL/SPIR-V and a pass to SSA, NIR still uses
   variables (`deref_var`, `load_deref`, and `store_deref`) heavily
 - to lower variables, one needs to lower derefs to appropricate intrinsics
   - "system value" variable derefs to `nir_intrinsic_load_<sv>`
   - "in" variable derefs to `nir_intrinsic_load_input` or others
   - "out" variable derefs to `nir_intrinsic_store_output` or others
   - "uniform" variable derefs to `nir_intrinsic_load_uniform`
 - however, those intrinsics require `driver_location` to be set
   - and?
 - freedreno tools
   - `ir3_compiler` to compile GLSL to IR3
   - `IR3_SHADER_DEBUG=disasm` to `nir_print_shader` final NIR and
     `ir3_shader_disasm` binary
 - freedreno
   - in CSO creation, `ir3_shader_from_nir` is called to `nir_lower_io` and
     `ir3_optimize_nir` final NIR
   - in `draw_vbo`, `ir3_shader_get_variant` is called to
     `ir3_compile_shader_nir` NIR to IR3 and `ir3_shader_assemble` IR3
