RenderDoc
=========

## Build

- `cmake -S. -Bout -GNinja -DCMAKE_BUILD_TYPE=Debug -DENABLE_QRENDERDOC=off -DENABLE_PYRENDERDOC=off`
- to enable `qrenderdoc`,
  - `apt install python3-dev qtbase5-dev libqt5svg5-dev libqt5x11extras5-dev`
- `ninja install`, or
  - edit `renderdoc_capture.json` to point to local `librenderdoc.so`
  - copy `renderdoc_capture.json` to `~/.local/share/vulkan/implicit_layer.d`

## `renderdoccmd`

- capture
  - `renderdoccmd capture -c <output-name> -w <executable> ...`
  - might want `-d` to specify the working dir
- replay
  - `renderdoccmd replay <capture.rdc>` just works
