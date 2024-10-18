Mesa PanVK Compiler
===================

## Environment Variables

- `PANVK_DEBUG`
  - `nir` dumps lowered nir before passing it to the backend compiler
  - `trace` implies `sync` and dumps all submits
- `BIFROST_MESA_DEBUG` is for the bifrost compiler
  - `msgs` Print debug messages
  - `shaders` Dump shaders in NIR and MIR
  - `shaderdb` Print statistics
  - `verbose` Disassemble verbosely
  - `internal` Dump even internal shaders
  - `nosched` Force trivial bundling
  - `nopsched` Disable scheduling for pressure
  - `inorder` Force in-order bundling
  - `novalidate` Skip IR validation
  - `noopt` Skip optimization passes
  - `noidvs` Disable IDVS
  - `nosb` Disable scoreboarding
  - `nopreload` Disable message preloading
  - `spill` Test register spilling
