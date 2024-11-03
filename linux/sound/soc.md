Kernel asoc
===========

## HW

- audio input path
  - mic converts sound wave to mic-level voltage
  - pre-amp raises mic-level voltage to line-level voltage
  - adc samples the line-level voltage
- audio output path
  - dac converts samples to line-level voltage
  - amp raises line-level voltage to headphone-level voltage
  - headphone converts headphone-level voltage to sound wave
- built-in mic path for laptops / phones
  - the built-in mic is typically a mems dmic
    - mems stands for Micro-Electro-Mechanical System, and is very small
    - dmic stands for digital mic, and has built-in pre-amp and adc
    - it seems to work autonomously and kernel uses a generic `dmic-codec`
      driver for it
  - the ap typically connects to the built-in mic over i2s to transfer the
    audio samples
    - a sck/bclk line (serial/bit clock)
    - a ws/lrclk/fs line (word select / left-right clock / frame sync)
    - a sd line (serial data)
- built-in speaker path for laptops / phones
  - the ap typically connects to two codecs over i2s and i2c
    - over i2s to transfer the audio samples
    - over i2c to program the codecs
    - there are two codecs for left and right channels respectively
  - each codec consists of a dac and an amp
- external headphone/mic
  - headphone plug
    - TS, with only tip (mono) and sleeve (ground)
    - TRS, with tip (left), ring (right), and sleeve (ground)
    - TRRS, with tip (left), ring 1 (right), ring 2 (mic+), and sleeve (mic-)
      - this is the headphone/mic combo type
  - headphone jack has 4 lines for TRRS plus a jack detection line
  - headphone codec: jack side
    - jack tip, for headphone left channel
      - it is the output of a dac
    - jack ring 1, for headphone right channel
      - it is the output of another dac
    - jack ring 2 and sleeve, for mic+ and mic-
      - they are the input of a adc
      - there are another 2 lines also connecting ring 2 and sleave, which
        detect whether the headphone has a combo mic or not
    - jack detection
  - headphone codec: ap side
    - usually over i2s
    - dac input line, for headphone
    - adc output line, for mic

## ASoC

- <https://docs.kernel.org/sound/soc/index.html>
  - a machine/board driver glues together platform drivers and codec drivers
  - a platform driver targets an soc cpu and must have no machine-specific
    code
  - a codec driver targets a codec and must have no machine-specific or
    platform-specific code
- a machine driver defines a `struct snd_soc_card`
  - the machine driver calls `devm_snd_soc_register_card` to register its
    `snd_soc_card`
- codec drivers are under `sound/soc/codecs`
  - each codec driver must define a `snd_soc_dai_driver`, which is a means to
    communicate with the codec
