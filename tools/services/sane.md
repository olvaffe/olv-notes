SANE
====

## Overview

- <http://www.sane-project.org/>
- `scanimage -L` lists devices
- `scanimage -d 'pixma:foo' -A` lists backend options
- `scanimage -d 'pixma:foo' --resolution 300 --mode Gray --format jpeg -o test.jpg`
  scans an image
  - `--resolution 300` sets the dpi to 300
  - `--mode Gray` sets the mode to gray
- troubleshooting
  - makes sure the user has permission to the raw usb device
  - might need to power cycle the scanner
  - `usblp` might conflict?
