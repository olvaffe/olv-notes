ARM Mali
========

## History

- 2007-: Utgard
  - Mali-{200,300,400,450}
  - GLES 2.0
  - vector ISA
- 2010: Midgard
  - Mali-T604
  - GLES 3.1
  - 4x or 16x FSAA
  - 1 to 4 unified shader cores
  - 2 ALUs per shader core
  - vector ISA
  - 1 MMU
  - 32KB to 256KB L2
  - 16x16 tiles
- 2011: Midgard
  - Mali-T658
  - GLES 3.1
  - more cores and L2
  - 4 ALUs per core
- 2012: Midgard
  - Mali-T{624,628,678}
  - GLES 3.1
  - ASTC
- 2013: Midgard
  - Mali-T{720,760}
  - GLES 3.2
  - 4x to 16x MSAA
  - 256KB to 2048MB L2
  - 1 to 16 cores
  - AFBC framebuffer compression
- 2015: Midgard
  - Mali-T{820,830,860}
- 2016 Bifrost
  - Mali-G{51,71}
  - 1 to 32 cores
  - scalar ISA
  - quad-based ALUs
  - full coherency
- 2017 Bifrost
  - Mali-G{52,72}
- 2018 Bifrost
  - Mali-G76
- 2019 Valhall
  - Mali-G{57,77}

## Midgard

- architecture
  - four shader cores
  - a MMU
  - a tiler
  - a job manager
  - a big L2 cache sitting in front of the main memory
- a shader core
  - supports vertex processing
  - supports rasterization
  - supports fragment processing
- 16x16 tiles
- 4KB on-chip memory

## Bifrost

- architecture
  - up to 32 shader cores
- a shader core
  - 
