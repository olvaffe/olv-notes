EC
==

## Directory Structure

- A board under `board/` decides the baseboard
- A baseboard under `baseboard/` decides the controller
- A controller under `chip/` decides the CPU under `core/`

## Build System

- To get ec.bin,
  - `$(out)/%.elf: $(out)/%.lds $(objs)`
    - use ec.lds to link
  - `$(out)/%.flat: $(out)/%.elf $(out)/%.smap $(build-utils)`
    - create EC binary image
  - `$(out)/$(PROJECT).obj: common/firmware_image.S $(out)/firmware_image.lds $(flat-y)`
    - create firmware image that combines several binary images
  - `$(out)/%.bin: $(out)/%.obj`
    - create firmware binary
