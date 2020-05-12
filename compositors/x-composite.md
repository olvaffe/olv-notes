X Composite Extension
=====================

## Accompanying extensions

- Shape Extension, for non-rectangular windows
  - each window has three regions
    - bounding region is the real-estate of the window in the parent window
    - clip region is the renderable region within the bounding region
      - the diff between bounding/clip regions is called the border, for
      	things such as window shadows
    - input region is the region that takes inputs
- Damage Extension, for dirty pixel tracking
  - we will need it to repaint a window
- XFIXES Extension, for misc core protocol fixes
  - we will need it to work with the regions of windows or damaged regions

## Overview

- the compositor calls `XCompositeRedirectSubwindows` on the root window
  - every child of the root window will render to its own off-screen storage
  - `XCompositeNameWindowPixmap` can be used to get the `Pixmap` for the
    off-screen storage
- whenever there is any damage in any window, repaint the damaged region
  - `XCreatePixmap` a new pixmap of the screen size
    - for double-buffering
  - for each window, render its off-screen pixmap and its shadow, etc, to the
    new pixmap.
  - copy from the new pixmap to the root window
- there can be other designs.  E.g., the compositor can repaint the entire
  back buffer and requests a page flip
