# Kernel asoc

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

## Example: 4-Speaker Laptop

- the 4 speakers are woofer and tweeter on left and right
- data flow: `app -> pipewire -> pcm -> dsp -> swr xN -> amp x4 -> speaker x4`
- dt describes a dai link for the speaker path
  - the link is a backend (covering swr, amp, speaker) and does not correspond
    to a user-visible pcm device
  - `platform` refers to dsp
  - `cpu` refers to dsp dma port
    - dsp outputs processed pcm with all 4 channels interleaved
  - `codec` refers to all codecs behind dsp
    - first codec reads data from dsp dma, post-processes, and generates
      bitstream
    - second codec is soundwire controller
    - third codec is amp
- dsp topology describes a graph for the speaker path
  - frontend subgraph: `read interleaved pcm from dram -> convert to internal format -> apply volume gain -> write to internal buffer`
    - the dsp firmware provides functions for high-level ops
    - each node in the subgraph refers to a dsp-specific function
    - it also tells alsa core to create a user-visible pcm device
      - it also specifies supported pcm format, such as `S16_LE`, 48khz, 2 or
        4 channels
  - backend subgraph: `read from internal buffer -> remap channels -> write to dma port`
  - mixer widget: `frontend subgraph -> backend subgraph`
    - a mixer is more like a switch in the topology; it can have N inputs and
      M outputs, but only one input and one output can be active at any time
- ucm config
  - a full pipeline looks like: `pcm dev -> dsp -> dai`
    - codec drivers for dai expose low-level codec hw controls as kcontrols
    - dsp topology can define kcontrols as well
    - `amixer -D hw:0 controls` lists all kcontrols
  - ucm groups low-level kcontrols to high-level kcontrols in userspace
    - e.g. each of the 4 amps has a hw volume kcontrol
      - ucm remaps them to a single volume kcontrol
    - `amixer -D default:0 controls` lists raw and remapped kcontrols
  - ucm also defines use cases
    - `_fboot` runs `FixedBootSequence`
      - it is run on every boot to init ucm state
    - `_boot` runs `BootSequence`
      - it is run on first boot to init kcontrols to default values
      - subsequent boots rely on save/restore
    - `_verb Foo` runs `SectionUseCase."Foo"`'s `EnableSequence` and the old
      verb's `DisableSequence`
      - `HiFi` is the most commonly verb, indicating media playback
    - `_enadev Foo` runs `SectionDevice."Foo"`'s `EnableSequence` and the
      disabled dev's `DisableSequence`
      - common devices are `Speaker`, `Mic`, `Headphones`, `Headset`, etc.
    - iow, play music on speaker is `_verb HiFi _enadev Speaker`

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
