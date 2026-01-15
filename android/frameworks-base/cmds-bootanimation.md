BootAnimation
=============

## Overview

- `BootAnimation::BootAnimation`
  - `SurfaceComposerClient::make` creates a `SurfaceComposerClient`
- `BootAnimation::readyToRun`
  - `SurfaceComposerClient::createSurface` creates a `SurfaceControl`
  - `SurfaceComposerClient::Transaction`
    - `set*` preps a transaction
    - `apply` applies a transaction
  - `SurfaceControl::getSurface` creates a `Surface`
  - `eglCreateWindowSurface` creates a `EGLSurface` from `Surface`
- `BootAnimation::threadLoop`
  - `loadAnimation` loads animation zipfile
  - `playAnimation` draws and `eglSwapBuffers` each animation frame
