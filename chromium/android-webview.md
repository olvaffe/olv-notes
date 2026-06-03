# Chromium WebView

## Perfetto

- there are app process and renderer process
- flow
  - app main: `Choreographer#doFrame`
  - renderer compositor: `Graphics.Pipeline STEP_GENERATE_COMPOSITOR_FRAME`
  - renderer compositor: `Graphics.Pipeline STEP_SUBMIT_COMPOSITOR_FRAME`
  - app `VizWebView`: `Graphics.Pipeline STEP_RECEIVE_COMPOSITOR_FRAME`
  - app `VizWebView`: `Graphics.Pipeline STEP_SURFACE_AGGREGATION`
  - app `VizWebView`: `Graphics.Pipeline STEP_SEND_BUFFER_SWAP`
  - app `RenderThread`: `Graphics.Pipeline STEP_BUFFER_SWAP_POST_SUBMIT`
  - app `RenderThread`: `Graphics.Pipeline STEP_FINISH_BUFFER_SWAP`
  - app `VizWebView`: `Graphics.Pipeline STEP_SWAP_BUFFERS_ACK`
- app `Chrome_InProcGpuThread` appears to be the "gpu process" for WebView
  - `RendererRasterWorker` is ui gpu job
  - `WebGL` is webgl gpu job
- app `RenderThread` from hwui
  - it renders the webview to the final buffer and swaps
