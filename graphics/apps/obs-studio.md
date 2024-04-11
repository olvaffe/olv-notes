OBS Studio
==========

## Build

- <https://github.com/obsproject/obs-studio/wiki/build-instructions-for-linux>
  - `git clone --recursive https://github.com/obsproject/obs-studio.git`
  - install dependencies
  - `cmake -S . -B out -G Ninja -DLINUX_PORTABLE=ON \
           -DENABLE_AJA=OFF -DENABLE_BROWSER=OFF \
           -DENABLE_NATIVE_NVENC=OFF -DENABLE_WEBRTC=OFF \
           -DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache`
  - `ninja -C out`
  - `ninja -C out install`
  - `cd out/install/bin/64bit`
  - `./obs`
- top-level options
  - `-DENABLE_UI=ON` enables ui
  - `-DENABLE_SCRIPTING=ON` enables scripting
  - `-DENABLE_HEVC=ON` enables hevc encoder
  - `-DLINUX_PORTABLE=OFF` disables portable packaging
  - `-DENABLE_PULSEAUDIO=ON` enables pulseaudio support
  - `-DENABLE_WAYLAND=ON` enables wayland support
- plugin options
  - `-DENABLE_PLUGINS=ON` enables plugins
  - `-DENABLE_AJA=ON` enables AJA hw support
  - `-DENABLE_DECKLINK=ON` enables DeckLink hw support
  - `-DENABLE_ALSA=ON` enables ALSA support
  - `-DENABLE_JACK=OFF` disables JACK support
  - `-DENABLE_PIPEWIRE=ON` enables pipewire support
  - `-DENABLE_V4L2=ON` enables V4L2 support
  - `-DENABLE_UDEV=ON` enables udev support for V4L2
  - `-DENABLE_BROWSER=ON` enables browser support
  - `-DENABLE_FFMPEG_LOGGING=OFF` disables logging for ffmpeg
  - `-DENABLE_NEW_MPEGTS_OUTPUT=ON` enables native SRT/RIST mpegts output for ffmpeg
  - `-DENABLE_NATIVE_NVENC=ON` enables nvenc support
  - `-DENABLE_FFMPEG_MUX_DEBUG=OFF` disables mux debug for ffmpeg
  - `-DENABLE_LIBFDK=OFF` disables FDK AAC support
  - `-DENABLE_VST=ON` enables VST plugin support
  - `-DENABLE_WEBRTC=ON` enables webrtc output support
  - `-DENABLE_WEBSOCKET=ON` enables websocket output support
  - `-DENABLE_OSS=ON` enables OSS audio support
  - `-DENABLE_SERVICE_UPDATES=ON` enables rtmp service update support
  - `-DENABLE_SNDIO=OFF` disables sndio audio support
  - `-DENABLE_FREETYPE=ON` enables freetype text support
  - `-DENABLE_VLC=ON` enables vlc support
