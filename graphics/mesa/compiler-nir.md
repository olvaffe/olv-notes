NIR
===

## Debug

- `MESA_SHADER_CACHE_DISABLE` to disable disk cache
  - otherwise, no shader compilation after the first run
  - it was called `MESA_GLSL_CACHE_DISABLE`
- `MESA_GLSL_VERSION_OVERRIDE` to override about GLSL version
- `MESA_GLSL` for various GLSL debug prints
  - `dump` to print
    - GLSL source, GLSL IR, and compiler errors on `glCompileShader`
    - GLSL IR, NIR, and compiler errors on `glLinkProgram`
    - ARB source and compiler errors on `glProgramStringARB`
  - `log` to write GLSL source to files in the working directory on
    `glCompileShader`
  - `source` to print GLSL source on `glCompileShader`
  - `errors` to print compiler errors on `glCompileShader` or `glLinkProgram`
    failures
- `NIR_DEBUG` for various NIR debug prints (only on debug builds!)
  - `tgsi` to print NIR and TGSI on conversions
  - `print` to print NIR after each pass
  - `print_*` instead to print NIR only for certain stages
  - `nir_print_shader`
    - `source_sha1` is the sha1 of the source code
      - only used by GLSL
    - `name` is the name of the shader
      - `GLSL%d` if GLSL shader
      - `ARB%d` if ARB program
      - `nir_builder_init_simple_shader` can set the name of internal shaders
      - not used otherwise
    - `label` is the debug label
      - only used by `glObjectLabel`

## Variables

- a `nir_variable` represents a variable in a high-level language
  - it encapsulates all info about the high-level variable, such as type,
    qualifiers, dimensions, storage, etc.
  - but it is not directly usable
- a `nir_deref_instr` results in a "pointer" to the variable
  - `dest` is a SSA `nir_dest`
  - `dest.ssa.parent_instr->var` points back to the variable
  - the dest is a pointer and is still not directly usable
- a `nir_intrinsic_instr` with op `nir_intrinsic_load_deref` loads the value
  - `instr->src[0].ssa` points to the "pointer"
  - `instr->dest.ssa` holds the variable value
  - `nir_variable_mode` specifies the mode/storage of the variable
  - `nir_lower_io` lowers `nir_intrinsic_load_deref` to one of low-level
    `nir_intrinsic_load_*` depending on the mode of the variable
- when the variable is more than a vec4, such as a matrix or an array or a
  struct, multiple `nir_deref_instr` are needed to get a `nir_dest` that can
  be used as a `nir_src` in `nir_intrinsic_load_deref`

## Instructions

- `nir_instr` is the base class of instructions
  - `struct exec_node node` is how the instr is managed by a block
  - `struct nir_block *block` is the block the instr belongs to
  - `nir_instr_type type` is one of
    - `nir_instr_type_alu`
    - `nir_instr_type_deref`
    - `nir_instr_type_call`
    - `nir_instr_type_tex`
    - `nir_instr_type_intrinsic`
    - `nir_instr_type_load_const`
    - `nir_instr_type_jump`
    - `nir_instr_type_undef`
    - `nir_instr_type_phi`
    - `nir_instr_type_parallel_copy`
  - `uint8_t pass_flags` is for arbitrary temp data
    - a pass can call `nir_shader_clear_pass_flags` to clear it to 0 and use
      it for any purpose
  - `uint32_t index` is the seqno of the instr
    - `nir_index_instrs` reindexes them
- each instruction has zero or more `nir_def` and zero or more `nir_src`
  - `nir_def` is def, aka ssa or dst
    - `nir_instr *parent_instr` is the instr that creates the def
      - being an ssa, each `nir_def` is created by some instr
    - `struct list_head uses` is the list of `nir_src` that references this ssa
      - when `nir_instr_insert` inserts an instr into a block, `add_defs_uses`
        adds the srcs to the list
    - `unsigned index` is the seqno of the ssa within the function
      - when `nir_instr_insert` inserts an instr into a block, `add_defs_uses`
        sets the seqno
      - `nir_index_ssa_defs` reindexes them
    - `uint8_t num_components` is the type width (`[1, NIR_MAX_VEC_COMPONENTS]`)
    - `uint8_t bit_size` is the type size (1, 8, 16, 32, or 64)
    - `bool divergent` is whether the ssa is divergent or uniform
      - `nir_divergence_analysis` sets it
  - `nir_src` is src
    - `uintptr_t _parent` is the instr that consumes the src
      - when `nir_instr_insert` inserts an instr into a block, `add_defs_uses`
        calls `nir_src_set_parent_instr` to init this
    - `struct list_head use_link` is managed by `nir_def::uses`
    - `nir_def *ssa` is the referenced `nir_def`
      - `nir_src_for_ssa` returns a `nir_src` that refers to the ssa
- life of an alu instruction
  - `nir_alu_instr_create` allocs a `nir_alu_instr`
  - `nir_def_init` inits `alu->def`
  - `alu->src[]` is initialized using `nir_src_for_ssa`
  - `nir_instr_insert` inserts the instr to `block->instr_list`
    - `nir_cursor_before_block` inserts to the head of the list
    - `nir_cursor_after_block` inserts to the tail of the list
    - `nir_cursor_before_instr` inserts before another instr in the list
    - `nir_cursor_after_instr` inserts after another instr in the list
    - `add_defs_uses` updates `src->_parent`, `src->use_link`, `def->uses`,
      and `def->index`
- when a pass replaces `old` by `new`
  - `nir_instr_insert_before(old, new)` inserts `new` before `old`
  - `nir_instr_remove(old)` removes `old`
    - `remove_defs_uses` loops through all old srcs and removes them from uses
    - the instr is then removed from the list
  - `nir_def_rewrite_uses(&old->def, &new->def)` calls `nir_src_rewrite` on
    all uses of `old->def`
    - if an src uses `old->def`, `nir_src_rewrite` remove the srct from
      `old->def.uses`, update `src->ssa` to `new->def`, and add src to
      `new->def.uses`
  - optional `nir_instr_free` to free the instr now
- `nir_alu_instr` has type `nir_instr_type_alu`
  - `nir_op op` is the alu op
    - `nir_op_mov`, `nir_op_fadd`, etc.
  - `bool exact` means any pass must ensure bit-for-bit identical val
  - `nir_def def` is the def
  - `nir_alu_src src[]` is the alu srcs
    - the array size is looked up from `nir_op_infos[op]`
    - `uint8_t swizzle[NIR_MAX_VEC_COMPONENTS]` is the swizzle
- a `nir_variable` should be derefed and loaded using special `nir_instr`s to
  create a `nir_dest`, before it is usable by normal `nir_instr`
- `nir_op`
  - opcode of `nir_alu_instr`
  - generated by and documented in `nir_opcodes.py`
- `nir_intrinsic_op`
  - opcode of `nir_intrinsic_instr`
  - generated by and documented in `nir_intrinsics.py`
    - `INTR_OPCODES`
- `nir_intrinsic_index_flag`
  - constants that can be associated with a `nir_intrinsic_instr`
  - generated by and documented in `nir_intrinsics.py`
    - `INTR_INDICES`

## Control Flows

- A `nir_function` represents a high-level function declaration.  It is not a
  `nir_cf_node`.  But it can have a `nir_function_impl` which is a
  `nir_cf_node`
- a `nir_cf_node` tree consists of
  - a `nir_function_impl` which is the root node
  - `nir_loop` and `nir_if` which are branch nodes
  - `nir_block` which are leaf nodes
  - the first and the last children of a root node or a branch node always
    exist and are always `nir_block`
    - `nir_function_impl_create_bare` adds a `nir_block` child
    - `nir_loop_create` adds a `nir_block` child
    - `nir_if_create` adds two `nir_block` children, for "then" and "else"
      respectively
    - a sibling node is added with `nir_cf_node_insert`, which always
      surrounds the node with `nir_block`
  - to go to the root node, `nir_cf_node_get_function`
  - to move between sibling nodes,
    - `nir_cf_node_next`
    - `nir_cf_node_prev`
    - `nir_cf_node_is_first`
    - `nir_cf_node_is_last`
- `nir_function_impl`
  - `body` is the list of the children nodes
  - `nir_function_impl_create_bare` always adds a start block as the first
    child
  - there is also a special end block that is not added to the tree
  - `nir_start_block` returns the start block
  - `nir_impl_last_block` returns the last block
    - which is not the special end block
- `nir_loop`
  - `body` is a list of the children nodes
  - `continue_list` is a list of continue targets
  - `nir_loop_create` always adds a block as the first child
- `nir_if`
  - `condition` determines which path to take
  - `then_list` and `else_list` are two lists of the children nodes
  - `nir_if_create` always adds a block to each list as the first children
- `nir_block`
  - `instr_list` is a list of instructions
  - all `nir_block` nodes in the tree also form a control flow graph, CFG
  - `predecessors` and `successors` are edges of the CFG

## SPIR-V to NIR

- GLSL
  - `in vec4 in_color;`
  - `out vec4 out_color;`
  - `void main() { out_color = in_color; }`
- `spirv_to_nir`
- NIR
  - `decl_var shader_in vec4 in_color (VARYING_SLOT_VAR0.xyzw, 0, 0)`
  - `decl_var shader_out vec4 out_color (FRAG_RESULT_DATA0.xyzw, 0, 0)`
  - `vec1 ssa0 = deref_var &in_color (shader_in vec4)`
  - `vec1 ssa1 = deref_var &out_color (shader_out vec4)`
  - `vec4 ssa2 = intrinsic load_deref(ssa0) (0/*access*/)`
  - `instrinsic store_deref(ssa1, ssa2) (15/*wrmask*/, 0/*access*/)`
- `NIR_PASS` is for normal passes
- `NIR_PASS_V` is for passes that do not return progress.  V stands for
  variant.
- compile
  - `nir_lower_io_to_temporaries` adds a temp var.  It replaces `store_deref`
    dest by a temp deref and insert a `copy_deref` from the temp deref to the
    real dest
  - `nir_lower_global_vars_to_local` moves the temp var into `main`
  - `nir_lower_vars_to_ssa` undoes `nir_lower_io_to_temporaries` but leaves
    some dead code
  - `nir_lower_alu_to_scalar` scalarizes a vec4 mov alu to 4 float mov alu
  - `nir_copy_prop` makes the mov alus dead code
  - `nir_opt_dce` removes the dead code
  - `nir_remove_dead_variables` removes the dead temp var
- the compiled NIR can be saved to disk cache
- link
  - `nir_lower_io_to_scalar_early` scalarizes `in_color`
  - `nir_link_opt_varyings` replaces `in_color.w` by 1.0
    - it knows VS writes 1.0 for W
  - `nir_lower_io_to_vector` undoes scalarization of `in_color`
     - it knows there is no need to load `in_color.w` now
- postprocess
  - lower inputs
    - for each input var,
      `var->data.driver_location = var->data.location`, where
      `location` is `VARYING_SLOT_VAR0`
    - `var->data.interpolation = INTERP_MODE_SMOOTH` unless flat shading or
      explicitly set
    - `nir_lower_io(nir, nir_var_shader_in, type_size_vec4, ...)` finds the
      `load_deref` and replaces it by
      - `load_const(0.0)`
      - `intrinsic load_barycentric_pixel`
      - `intrinsic load_interpolated_input` w/ base set to
      	`var->data.driver_location`
  - lower outputs
    - for each output var,
      `var->data.driver_location = var->data.location`
    - `nir_lower_io(nir, nir_var_shader_out, type_size_vec4, ...)` finds the
      `store_deref` and replaces it by
       - `load_const(0.0)`
       - `intrinsic store_output` w/ base set to `var->data.driver_location`
  - `nir_opt_dce` removes dead code
  - `nir_opt_cse` removes common subexpressions
    - e.g., removes all but one `load_const(0.0)`
- final NIR
  - `vec2 ssa0 = intrinsic load_barycentric_pixel () (/*interp_mode*/1)`
  - `vec1 ssa1 = load_const (0.0f)`
  - `vec3 ssa2 = intrinsic load_interpolated_input (ssa0, ssa1) (/*base*/32, /*component*/0)`
  - `vec1 ssa3 = load_const (1.0f)`
  - `vec4 ssa4 = vec4 ssa2.x, ssa2.y, ssa2.z, ssa3`
  - `intrinsic store_output (ssa4, ssa1) (/*base*/8, /*wrmast*/15, /*component*/0, /*type*/160)`
- codegen
- UBOs
  - `layout(set = 0, binding = 0) uniform xxx { float qqq; };`
  - `decl_var ubo INTERP_MODE_NONE xxx  (~0, 0, 0)`
    - the three numbers are `location`, `driver_location`, and `binding`
  - `vec1 32 ssa_2 = load_const (0x00000000 = 0.000000)`
  - `vec3 32 ssa_3 = intrinsic vulkan_resource_index (ssa_2) (desc_set=0, binding=0, desc_type=UBO /*6*/)`
    - at `desc_set` and `binding` is an array of resources
    - `ssa_2` is the array index
  - `vec3 32 ssa_4 = intrinsic load_vulkan_descriptor (ssa_3) (desc_type=UBO /*6*/)`
    - this loads the resource descriptor from the resource index, which is
      usually no-op
  - `vec3 32 ssa_5 = deref_cast (xxx *)ssa_4 (ubo xxx)  /* ptr_stride=0, align_mul=0, align_offset=0 */`
  - `vec3 32 ssa_6 = deref_struct &ssa_5->qqq (ubo float) /* &((xxx *)ssa_4)->qqq */`
  - `vec1 32 ssa_7 = intrinsic load_deref (ssa_6) (access=0)`

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

## Common NIR Lowerings

- common lowering coming out of high-level language
  - the main goal is to convert variable loads/stores to ssas for
    optimizations
  - `nir_opt_reuse_constants` deduplicates and moves all `load_const` to the
    first block
  - `nir_inline_functions` inlines all functions into their callers
  - `nir_lower_global_vars_to_local` lowers global variables to local
    variables when they are used by a single function
  - `nir_lower_vars_to_ssa` lowers local variables to ssa
    - this replaces local variable loads/stores by phis
  - `nir_lower_system_values` lowers sysval loads to intrinsics
    - e.g., `SYSTEM_VALUE_GLOBAL_INVOCATION_ID` var load is replaced by
      `load_global_invocation_id` intrinsic
  - `nir_lower_compute_system_values` lowers compute sysval loads
    - e.g., `load_global_invocation_id` intrinsic is replaced by
      `load_workgroup_id`, `load_local_invocation_id`, and alus
  - `nir_remove_dead_variables` removes dead variables
  - `nir_lower_load_const_to_scalar` lowers vector `load_const` to scalar
    `load_const`
    - e.g., a const load of vec3 is replaced by 3 const loads of float
      followed by a `nir_op_vec3`
- common optimizations coming out of high-level language
  - these are applied repeatedly in a loop
    - `nir_lower_alu_width` replaces vector alus by scalar alus
      - e.g., a iadd of vec3 is replaced by 3 iadd of float followed by a
        `nir_op_vec3`
    - `nir_lower_phis_to_scalar` replaces vector phis by scalar phis
      - e.g., a phi of vec3 is replaced by
        - 3 phis of float and a vec3
        - 3 movs at the end of the two predecessors respectively
    - `nir_copy_prop` performs copy propogation
      - if an instruction is mov, replace all uses of its def by its src
      - movs created by `nir_lower_phis_to_scalar` above are usually removed as
        a result
    - `nir_opt_dce` performs dead code elimination
    - `nir_opt_loop` eliminates unnecessary loop blocks
    - `nir_opt_if` optimizes if+loop blocks
    - `nir_opt_cse` performs common subexpression elimination
      - if two instructions produce the same value, eliminate all but one
    - `nir_opt_peephole_select` replaces phis by bcsels when cheap
    - `nir_opt_constant_folding` performs constant folding
      - it evalutes constant expressions at compile time
      - it is most effective after copy propagation
    - `nir_opt_algebraic` matches and replaces instructions based on the rules
      defined in `nir_opt_algebraic.py`
    - `nir_opt_loop_unroll` unrolls simple loops
  - these are applied once
    - `nir_opt_move` moves defs to just before their first uses within the
      same block
      - this can reduce reg pressure
- 2nd round of lowering coming out of high-level language
  - the main goal is to lower to instructions that are closer-to-hw
  - `nir_opt_access` infers readonly, writeonly, etc.
  - `nir_lower_explicit_io` lowers loads of explicitly-laid-out buffers (ubo,
    ssbo, push consts, etc.) by intrinsics in addrs and offsets
    - load of `nir_var_mem_push_const` is replaced by `load_push_constant`
    - load of `nir_var_mem_ubo` is replaced by `load_global_constant*` or
      `load_ubo` depending on the addr format
    - load of `nir_var_mem_ssbo` is replaced by `load_global` or `load_ssbo`
    - store of `nir_var_mem_ssbo` is replaced by `store_global*` or `store_ssbo`
- 2nd round of optimizations coming out of high-level language
  - same as the first round after 2nd round of lowering
- common lowering before passing nir to backend
  - this is more hw-dependent
  - `nir_opt_load_store_vectorize` coalesces loads/stores
    - e.g., 3 loads to consecutive floats can be replaced by a load to 3
      floats
  - `nir_vk_lower_ycbcr_tex` emulates ycbcr sampling
  - a backend-specific pass applies vk pipeline layout
    - e.g., the generic `vulkan_resource_index`, `load_vulkan_descriptor`, and
      `store_ssbo` may be replaced by backend-specific intrinsics to
      - load the descriptor set address
      - calculate the descriptor offset in the descriptor set
      - load the descriptor from the descriptor set
      - store via the descriptor
  - `nir_opt_sink` moves defs to the least common ancestor block of consuming
    instructions
    - this is different from `nir_opt_move`, which moves within the same block
  - `nir_lower_int64` emulates 64-bit ints
  - `nir_opt_idiv_const` and `nir_lower_idiv` emulate integer division
- common optimizations before passing nir to backend
  - some are the same as the prior optimizations; the new ones are
  - `nir_opt_offsets` performs constant folding for load/store offset
    - load/store intrinsics supports offset in src as well as a compile-time
      base; this pass folds the compile-time const from src to base
  - `nir_opt_algebraic_late` uses another set of rules defined in
    `nir_opt_algebraic.py`
- 2nd round of lowering/optimizations before passing nir to backend
  - some are the same as the prior optimizations; the new ones are
  - `nir_lower_fp16_casts` emulates casts between fp16/fp32/fp64
  - `nir_opt_vectorize` vectorizes alus but seems limited
    - e.g., 3 fadds for xyz components of the same ssa respectively are
      replaced by 1 fadd of 3 components

## NIR to backend

- after translation from GLSL/SPIR-V and a pass to SSA, NIR still uses
  variables (`deref_var`, `load_deref`, and `store_deref`) heavily
- to lower variables, one needs to lower derefs to appropricate intrinsics
  - "system value" variable derefs to `nir_intrinsic_load_<sv>`
  - "in" variable derefs to `nir_intrinsic_load_input` or others
  - "out" variable derefs to `nir_intrinsic_store_output` or others
  - "uniform" variable derefs to `nir_intrinsic_load_uniform`
    - modern hw has no special storage for uniforms
    - `nir_lower_uniforms_to_ubo` lowers `nir_intrinsic_load_uniform` to
      `nir_intrinsic_load_ubo`
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

## IO

- examples
  - loading `in vec4 in_val` is translated to a `deref_var` and a `load_deref`
  - loading `in centroid vec4 in_val` or `interpolateAtCentroid` is translated
    to a `deref_var` and a `interp_deref_at_centroid`
- `nir_lower_io`
  - `load_deref` is lowered to
    - vs: `load_input`
    - fs: `load_barycentric_pixel` and `load_interpolated_input`
  - `interp_deref_at_centroid` is lowered to
    - fs: `load_barycentric_centroid` and `load_interpolated_input`
- `nir_assign_io_var_locations`
  - it scans variables and assigns `var->data.driver_location`
  - callers use this to update `nir->num_inputs` / `nir->num_outputs` as well
- fs inputs
  - barycentric coordinates
  - `load_barycentric_pixel` loads the barycentric weight vector for the
    fragment
  - `load_interpolated_input` loads the interpolated value of an input
    - src0 is the barycentric weight vector
    - src1 is offset which is always 0?

## Varyings

- vulkan requires `layout(location)`, aka `explicit_location`
  - the locations of varyings are explicitly assigned by users
  - drivers tend to just do standard optimizations
    - `nir_link_opt_varyings` optmizes out unecessary varyings
      - a varying whose value is a const
      - a varying whose value is a uniform
      - two varying having the same value
    - `nir_remove_dead_variables` removes dead variables
    - `nir_remove_unused_varyings` removes varyings that are unused
    - `nir_compact_varyings` modifies varying `location` and `location_frac`
- gl does not require `layout(location)`
  - `gl_nir_link_varyings` links varyings by assigning `location` and
    `location_frac` when they are not explicitly specified
    - `assign_initial_varying_locations`
      - it uses name matching or explicit location to link varyings
      - `varying_matches_assign_temp_locations` assigns a unique location (from
        `VARYING_SLOT_VAR0` on) for each varying
    - `link_shader_opts`
      - among others, `nir_lower_io_to_scalar_early` scalarizes varyings
    - `remove_unused_shader_inputs_and_outputs`
    - `assign_final_varying_locations`
      - `varying_matches_store_locations` can assign variables in the same
        packing class to the same locations
      - `gl_nir_lower_packed_varyings` is a pass that makes sure `location_frac`
        is 0 unless it is explicitly specified
        - this is done becuase not all backend supports `location_frac`
  - in `st_link_nir`,
    - `nir_compact_varyings`
    - `st_nir_vectorize_io` re-vectorizes the varyings

## Optimizations

- `nir_opt_load_store_vectorize`
  - it looks at all loads/stores and tries to combine them
    - e.g., two consecutive 16-bit loads can potentially be combined into a
      single 32-bit load
  - `create_entry` creates an entry for each load/store
  - `vectorize_entries` sorts entries by offsets and tries to combine them
    - `sort_entries` sorts them
    - `vectorize_sorted_entries` combines them
  - `try_vectorize` tries to comine `high` into `low`
    - `new_size` is larger enough to cover both `low` and `high`
    - `new_bitsize_acceptable` checks if `low` can use a new bit size
      - a backend-provided callback is used, to see if `low` can use the new
        bit size and the new component count, given the `align_mul` and
        `align_offset` requirement
  - `align_mul` and `align_offset` are defined and documented in
    `nir_intrinsics.py`
    - for example, a 16-bit load might have 2 for `align_mul` and 0 for
      `align_offset`
    - if a backend requires a 32-bit load to be 4-byte aligned, two such
      16-bit loads cannot be combined
    - what is `align_offset` for?
      - for a member of a struct, `align_mul` is the size of the struct and
        `align_offset` is the offset of the member
      - a misaligned `uint32_t` array could have `align_mul` 4 and
        `align_offse` 2
- `nir_opt_if`
  - `opt_split_alu_of_phi`

## `nir_opt_algebraic`

- `nir_opt_algebraic.py` generates `nir_opt_algebraic` variants
  - `nir_algebraic.AlgebraicPass` generates 4 variants
  - `nir_opt_algebraic` uses `optimizations`
  - `nir_opt_algebraic_before_ffma` uses `before_ffma_optimizations`
  - `nir_opt_algebraic_late` uses `late_optimizations`
  - `nir_opt_algebraic_distribute_src_mods` uses `distribute_src_mods`
- `nir_opt_algebraic` and `nir_opt_algebraic_late` are generic
  - `nir_opt_algebraic` should be called as a part of the regular optimization
    loop; e.g., after `spirv_to_nir` and regular lowering
  - `nir_opt_algebraic_late` should be called before codegen
- each optimization replaces a single instruction
  - the search expr can match multiple instructions, but only the outer most
    instruction is replaced
    - dce will get rid of the inner instructions
  - the replace expr can replace the single instruction by one or more
    instructions

## Subgroup

- `GL_ARB_shader_group_vote`
  - `nir_intrinsic_vote_any`
  - `nir_intrinsic_vote_all`
  - `nir_intrinsic_vote_feq`
  - `nir_intrinsic_vote_ieq`
- `GL_ARB_shader_ballot`
  - `nir_intrinsic_load_subgroup_size`
  - `nir_intrinsic_load_subgroup_eq_mask`
  - `nir_intrinsic_load_subgroup_ge_mask`
  - `nir_intrinsic_load_subgroup_gt_mask`
  - `nir_intrinsic_load_subgroup_le_mask`
  - `nir_intrinsic_load_subgroup_lt_mask`
  - `nir_intrinsic_ballot`
  - `nir_intrinsic_ballot_bitfield_extract`
  - `nir_intrinsic_ballot_bit_count_reduce`
  - `nir_intrinsic_ballot_find_lsb`
  - `nir_intrinsic_ballot_find_msb`
  - `nir_intrinsic_ballot_bit_count_exclusive`
  - `nir_intrinsic_ballot_bit_count_inclusive`
  - `nir_intrinsic_read_invocation`
  - `nir_intrinsic_read_first_invocation`
- `GL_KHR_shader_subgroup`
  - `nir_intrinsic_elect`
  - `nir_intrinsic_shuffle`
  - `nir_intrinsic_shuffle_xor`
  - `nir_intrinsic_shuffle_up`
  - `nir_intrinsic_shuffle_down`
  - `nir_intrinsic_quad_broadcast`
  - `nir_intrinsic_quad_swap_horizontal`
  - `nir_intrinsic_quad_swap_vertical`
  - `nir_intrinsic_quad_swap_diagonal`
  - `nir_intrinsic_reduce`
  - `nir_intrinsic_inclusive_scan`
  - `nir_intrinsic_exclusive_scan`

## int64

- `nir_lower_int64`
- `should_lower_int64_instr` returns true when the instruction is an alu
  involving 64-bit integers and its lowering is enabled in
  `nir_lower_int64_options`
  - for `nir_op_i2i*` and `nir_op_u2*`, true if the src is 64-bit
  - for `nir_op_bcsel`, true if the two branches are 64-bit
  - for `nir_op_i{eq,ne,lt,...}` and `nir_op_u{eq,ne,lt,...}`, true if the
    two srcs are 64-bit
  - for `nir_op_find_lsb` and `nir_op_bitcount`, true if the src is 64-bit
  - for `nir_op_amul`, true if the dst is 64-bit
  - for the rest alu, true if the dst is 64-bit
    - because the rest of the instructions does not change integer widthds and
      it suffices to check the dst
- setting `nir_lower_mov64` enables lowering of these instructions
  - these instructions might have 64-bit src
    - `nir_op_i2i*` is lowered by `lower_i2i*`
    - `nir_op_u2u*` is lowered by `lower_u2u*`
    - `nir_op_i2f*` and `nir_op_u2f*` are lowered by `lower_2f`
  - these instructions have or might have 64-bit dst
    - `nir_op_bcsel` is lwoered by `lower_bcsel64`
    - `nir_op_b2i64` is lowered by `lower_b2i64`
    - `nir_op_f2i64` and `nir_op_f2u64` is lowered by `lower_f2`
  - `lower_i2i*` and `lower_u2u*` uses `nir_unpack_64_2x32_split_x` to unpack
    a 64-bit int to 2 32-bit ints and returns the lower 32-bit one
  - `lower_bcsel64` unpacks both branches, does two `nir_bcel`, and repacks
  - `lower_b2i64` uses `nir_pack_64_2x32_split` to pack the bool and zero as
    64-bit int
  - `lower_2f` is very complex
  - `lower_f2` uses `nir_ftrunc`, `nir_fdiv`, `nir_frem`, etc.

## `nir_tex_instr`

- `enum nir_texop`
  - `nir_texop_tex` and `nir_texop_txb`
    - glsl: `texture`
    - spirv:
      - `SpvOpImageSampleImplicitLod`
      - `SpvOpImageSparseSampleImplicitLod`
      - `SpvOpImageSampleDrefImplicitLod`
      - `SpvOpImageSparseSampleDrefImplicitLod`
      - `SpvOpImageSampleProjImplicitLod`
      - `SpvOpImageSampleProjDrefImplicitLod`
    - `tex` or `txb` depends on whether a bias is specified
  - `nir_texop_txl` and `nir_texop_txd`
    - glsl: `textureLod` and `textureGrad`
    - spirv:
      - `SpvOpImageSampleExplicitLod`
      - `SpvOpImageSparseSampleExplicitLod`
      - `SpvOpImageSampleDrefExplicitLod`
      - `SpvOpImageSparseSampleDrefExplicitLod`
      - `SpvOpImageSampleProjExplicitLod`
      - `SpvOpImageSampleProjDrefExplicitLod`
      - `txl` or `txd` depends on whether an lod or a grad is specified
  - `nir_texop_txf` and `nir_texop_txf_ms`
    - glsl: `texelFetch`
    - spirv:
      - `SpvOpImageFetch`
      - `SpvOpImageSparseFetch`
    - `txf` or `txf_ms` depends on MSAA
  - `nir_texop_txf_ms_fb`
    - this is generated by `nir_lower_fb_read` for advanced blending
  - `nir_texop_txf_ms_mcs_intel`
    - this is generated by intel to fetch mcs
  - `nir_texop_txs`
    - glsl: `textureSize`
    - spirv:
      - `SpvOpImageQuerySizeLod`
      - `SpvOpImageQuerySize`
  - `nir_texop_lod`
    - glsl: `textureQueryLOD`
    - spirv: `SpvOpImageQueryLod`
  - `nir_texop_tg4`
    - glsl: `textureGather`
    - spirv:
      - `SpvOpImageGather`
      - `SpvOpImageSparseGather`
      - `SpvOpImageDrefGather`
      - `SpvOpImageSparseDrefGather`
  - `nir_texop_query_levels`
    - glsl: `textureQueryLevels`
    - spirv: `SpvOpImageQueryLevels`
  - `nir_texop_texture_samples`
    - glsl: `textureSamples`
    - spirv: `SpvOpImageQuerySamples`
  - `nir_texop_samples_identical`
    - glsl: `textureSamplesIdenticalEXT`
  - `nir_texop_tex_prefetch`
    - this is generated by `ir3_nir_lower_tex_prefetch`
  - `nir_texop_fragment_fetch_amd`
    - spirv: `SpvOpFragmentFetchAMD`
    - this is also generated by amd to fetch fragment
  - `nir_texop_fragment_mask_fetch_amd`
    - spirv: `SpvOpFragmentMaskFetchAMD`
    - this is also generated by amd to fetch fmask
  - `nir_texop_descriptor_amd`
    - this is generated by `ac_nir_lower_resinfo` and
      `ac_nir_lower_image_opcodes` to return the image descriptor
  - `nir_texop_sampler_descriptor_amd`
    - this is generated by `ac_nir_lower_image_opcodes` to return the sampler
      descriptor
  - `nir_texop_lod_bias_agx`
    - this is generated by `agx_nir_lower_texture_early` to return the bias
  - `nir_texop_hdr_dim_nv`
    - this is generated by `nak_nir_lower_tex` to return the dim in header
  - `nir_texop_tex_type_nv`
    - this is generated by `nak_nir_lower_tex` to return the type
- `enum nir_tex_src_type`
  - `nir_tex_src_coord` is the texcoords
  - `nir_tex_src_projector` is the projector
    - this is available for `SpvOpImageSampleProj*`
  - `nir_tex_src_comparator`
    - this is available for `SpvOpImage*Dref*`
  - `nir_tex_src_offset`
    - this is from `SpvImageOperandsOffsetMask` or
      `SpvImageOperandsConstOffsetMask`
  - `nir_tex_src_bias`
    - this is from `SpvImageOperandsBiasMask`, which is valid for
      - `nir_texop_txb`
      - `nir_texop_tg4` (`SPV_AMD_texture_gather_bias_lod`)
  - `nir_tex_src_lod`
    - this is from `SpvImageOperandsLodMask`, which is valid for
      - `nir_texop_txl`
      - `nir_texop_txf`
      - `nir_texop_txs`
      - `nir_texop_tg4` (`SPV_AMD_texture_gather_bias_lod`)
  - `nir_tex_src_min_lod`
    - this is from `SpvImageOperandsMinLodMask`, which is valid for
      - `nir_texop_tex`
      - `nir_texop_txb`
      - `nir_texop_txd`
    - this is for sparse texturing, see `GL_ARB_sparse_texture_clamp`
  - `nir_tex_src_ms_index`
    - this is from `SpvImageOperandsSampleMask`, which is valid for
      - `nir_texop_txf_ms`
      - `nir_texop_fragment_fetch_amd`
  - `nir_tex_src_ms_mcs_intel`
    - this is valid for `nir_texop_txf_ms`
  - `nir_tex_src_ddx` and `nir_tex_src_ddy`
    - these are from `SpvImageOperandsGradMask`, which is valid for
      `nir_texop_txd`
  - `nir_tex_src_texture_deref` is tex var deref
    - this is availabe for all texops
  - `nir_tex_src_sampler_deref` is sampler var deref
    - this is availabe for texops that need require a sampler
  - `nir_tex_src_texture_offset`
    - this is from lowering `nir_tex_src_texture_deref`
    - e.g.,
      - `nir_tex_src_texture_deref` derefs a high-level var
      - the high-level var has `set = s` and `binding = b`
      - for non-bindless hw, that allows the backend compiler to determine the
        "offset" of the descriptor in the binding table
        - the deref is lowered to `nir_tex_src_texture_offset`
      - for bindless hw, that allows the backend compiler to load the
        descriptor from the descriptor set
        - the deref is lowered to `nir_tex_src_texture_handle`
  - `nir_tex_src_sampler_offset`
    - this is from lowering `nir_tex_src_sampler_deref`
  - `nir_tex_src_texture_handle`
    - this is from lowering `nir_tex_src_texture_deref`
  - `nir_tex_src_sampler_handle`
    - this is from lowering `nir_tex_src_sampler_deref`
  - `nir_tex_src_plane`
    - this is for ycbcr sampling
  - `nir_tex_src_backend1`
  - `nir_tex_src_backend2`
- helper functions
  - some texops need sampler and some don't
    - `nir_tex_instr_need_sampler`
  - some texops access image memory and some don't
    - `nir_tex_instr_is_query`
  - some texops require implicit derivs and some don't
    - `nir_tex_instr_has_implicit_derivative`
    - `nir_shader_supports_implicit_lod`

## Push Constants

- glsl: `layout(push_constant) uniform CONSTS { uint repeat; } consts;`
- spirv
  - `%CONSTS = OpTypeStruct %uint`
    - this declares a struct type
  - `%_ptr_PushConstant_CONSTS = OpTypePointer PushConstant %CONSTS`
    - this declares a pointer type (to the struct)
  - `%consts = OpVariable %_ptr_PushConstant_CONSTS PushConstant`
    - this defines a variable (which is a pointer to the struct)
  - `%_ptr_PushConstant_uint = OpTypePointer PushConstant %uint`
    - this declares a pointer type (to an uint)
  - `%42 = OpAccessChain %_ptr_PushConstant_uint %consts %int_0`
    - this defines a variable (which is a pointer to the struct member)
  - `%43 = OpLoad %uint %42`
    - this loads from the var
- nir
  - `decl_var push_const INTERP_MODE_NONE none CONSTS consts`
    - this defines the variable (of the struct)
  - `32    %26 = deref_var &consts (push_const CONSTS)`
    - this derefs the variable
  - `32    %27 = deref_struct &%26->repeat (push_const uint)  // &consts.repeat`
    - this derefs the struct member
  - `32    %28 = @load_deref (%27) (access=none)`
    - this loads from the member
- lowered nir
  - `32    %14 = @load_push_constant (%15 (0x0)) (base=0, range=4, align_mul=256, align_offset=0)`
    - the address format must be `nir_address_format_32bit_offset`

## SSBO

- glsl: `layout(set = 0, binding = 0) buffer DST { float data[]; } dst;`,
- spirv
  - `%_runtimearr_float = OpTypeRuntimeArray %float`
    - this declares an array type, whose length is unknown
  - `%DST = OpTypeStruct %_runtimearr_float`
    - this declares a struct type
  - `%_ptr_Uniform_DST = OpTypePointer Uniform %DST`
    - this declares a pointer type (to the struct)
  - `%dst = OpVariable %_ptr_Uniform_DST Uniform`
    - this defines a variable (which is a pointer to the struct)
  - `%_ptr_Uniform_float = OpTypePointer Uniform %float`
    - this declares a pointer type (to a float)
  - `%59 = OpAccessChain %_ptr_Uniform_float %dst %int_0 %int_0`
    - this defines a variable (which is a pointer to the first elem of the struct member)
  - `OpStore %59 %float_0`
    - this stores 0.0f to the var
- nir
  - `decl_var ssbo INTERP_MODE_NONE restrict DST dst (~0, 0, 0)`
    - this defines the variable (of the struct)
  - `32x3  %45 = @vulkan_resource_index (%16 (0x0)) (desc_set=0, binding=0, desc_type=SSBO)`
    - this defines the handle to the vulkan resource
  - `32x3  %46 = @load_vulkan_descriptor (%45) (desc_type=SSBO)`
    - this loads the vulkan resource descriptor
  - `32x3  %47 = deref_cast (DST *)%46 (ssbo DST)  (ptr_stride=0, align_mul=0, align_offset=0)`
    - this derefs the vulkan resource descriptor
  - `32x3  %48 = deref_struct &%47->data (ssbo float[])  // &((DST *)%46)->data`
    - this derefs the struct member
  - `32x3  %50 = deref_array &(*%48)[0] (ssbo float)  // &((DST *)%46)->data[0]`
    - this derefs the array elem
  - `@store_deref (%50, %16 (0.000000)) (wrmask=x, access=none)`
    - this stores 0.0 to the array elem
- lowered nir
  - `32x3  %1 = @vulkan_resource_index (%0 (0x0)) (desc_set=0, binding=0, desc_type=SSBO)`
  - `32x3  %2 = @load_vulkan_descriptor (%1) (desc_type=SSBO)`
    - with `nir_address_format_vec2_index_32bit_offset`, the result consists
      of 64-bit handle and 32-bit offset
  - `32    %3 = mov %2.z`
  - `32x2  %4 = vec2 %2.x, %2.y`
  - `@store_ssbo (%0 (0x0), %4, %3) (wrmask=x, access=writeonly, align_mul=1073741824, align_offset=0)`
    - this stores 0 to the descriptor/offset

## Compute

- glsl: `gl_GlobalInvocationID.x`
- spirv
  - `OpDecorate %gl_GlobalInvocationID BuiltIn GlobalInvocationId`
    - this decorates the variable (that haven't been defined yet)
  - `%v3uint = OpTypeVector %uint 3`
    - this declares vector type, uvec3
  - `%_ptr_Input_v3uint = OpTypePointer Input %v3uint`
    - this declares a pointer type (to uvec3)
  - `%gl_GlobalInvocationID = OpVariable %_ptr_Input_v3uint Input`
    - this defines a variable (which is a pointer to uvec3)
  - `%_ptr_Input_uint = OpTypePointer Input %uint`
    - this declares a pointer type (to uint)
  - `%15 = OpAccessChain %_ptr_Input_uint %gl_GlobalInvocationID %uint_0`
    - this defines a variable (which is a pointer to the first member of the
      variable)
  - `%16 = OpLoad %uint %15`
    - this loads from the struct member
- nir
  - `decl_var system INTERP_MODE_NONE none uvec3 gl_GlobalInvocationID (SYSTEM_VALUE_GLOBAL_INVOCATION_ID)`
    - this defines the variable
  - `32     %0 = deref_var &gl_GlobalInvocationID (system uvec3)`
    - this derefs the variable
  - `32x3   %3 = @load_deref (%0) (access=none)`
    - this loads from the variable
  - `32     %4 = mov %3.x`
    - this moves out the x member
- lowered nir
  - `32x3   %1 = @load_workgroup_id`
    - this loads the workgroup id (`gl_WorkGroupID`)
  - `32x3   %2 = @load_local_invocation_id`
    - this loads the local id (`gl_LocalInvocationID`)
  - `32     %4 = ishl %1.x, %3 (0x6)`
    - because of `layout(local_size_x = 64) in;`, this left-shifts the
      workgroup id by 6
  - `32     %5 = iadd %4, %2.x`
    - this sums both up
