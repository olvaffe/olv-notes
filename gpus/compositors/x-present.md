X Present Extension
===================

## Overview

- modesetting driver calls `present_screen_init` to enable screen-flip present
  support
  - only full-screen windows can be flipped
  - the rests are copied
- xwayland calls `present_wnmd_screen_init` to enable window-flip present
  support
  - all windows flip
