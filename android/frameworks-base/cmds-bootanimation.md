# BootAnimation

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

## Init

- `BootAnimation::BootAnimation`
- `BootAnimation::onFirstRef`
  - `preloadAnimation` loads zipfile
    - `findBootAnimationFile` inits `mZipFileName`
      - search paths
        - `/apex/com.android.bootanimation/etc/bootanimation.zip`
        - `/product/media/bootanimation.zip`
          - `ro.product.bootanim.file` can customize
        - `/oem/media/bootanimation.zip`
        - `/system/media/bootanimation.zip`
    - `loadAnimation` inits `Animation`
      - `parseAnimationDesc` parses `desc.txt`
        - `<width> <height> <fps>` is trivial
        - `c <repeat-count> <pause-between-repeat> <part-path>` defines a part
          - `c` means `playUntilComplete`
          - `part.backgroundColor` defaults to black
      - `preloadZip` iterates the zipfile
        - each png within a part defines a frame
        - part `trim.txt` defines frame x, y, w, h

## Run

- `BootAnimation::initDisplaysAndSurfaces`
  - it connects to sf and creates a surface
- `BootAnimation::initShaders`
  - `mImageShader`
  - `mTextShader`
- `BootAnimation::movie`
  - `playAnimation` plays the animation part by part
    - remember that
      - `part.count` is the repeat count
      - `part.frames.size()` is the frame count
    - for each part,
      - `shouldStopPlayingPart` decides if the part should end early
      - for each frame within the part,
        - `initTexture` loads a png
          - linear filtering is enabled
          - texobj 0 is replaced unless the frame can be repeated
        - it draws the png to each display
          - `projectSceneToWindow` inits viewport and scissor
          - `glCear` clears to `part.backgroundColor`
          - `glDrawArrays` draws a rectangle with `mImageShader`
          - if any text (clock, progress, etc.), it draws with `mTextShader`
          - `eglSwapBuffers` swaps
        - it sleeps for the fps target
