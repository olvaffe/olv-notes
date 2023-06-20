Kernel ALSA
===========

## ASoC

- <https://docs.kernel.org/sound/soc/index.html>
  - a machine/board driver glues together platform drivers and codec drivers
  - a platform driver targets an soc cpu and must have no machine-specific
    code
  - a codec driver targets a codec and must have no machine-specific or
    platform-specific code
