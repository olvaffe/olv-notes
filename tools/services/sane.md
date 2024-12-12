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

## Share a Local Scanner

- on the server,
  - edit `/etc/sane.d/saned.conf` to add allowed subnets
  - `sudo systemctl enable saned.socket`
- on the client,
  - edit `/etc/sane.d/dll.conf` to make sure `net` is enabled
  - edit `/etc/sane.d/net.conf` to add the ip of the server
  - `scanimage -L` should list the remote scanner

## Driverless Scanning

- protocols
  - Apple AirScan uses Mopria eSCL
  - Microsoft WSD Scan uses its own protocol
- on the client,
  - <https://github.com/alexpevzner/sane-airscan>
    - this is a sane backend that talks eSCL and WSD
    - it discovers `_uscan(s)._tcp` and `_scanner._tcp` devices automatically
  - there is also `sane-escl` backend which is less compatible
- to emulate an eSCL scanner on the server,
  - <https://github.com/SimulPiscator/AirSane>
    - this is a sane frontend that
      - advertises an eSCL service for clients
      - scans using sane
    - `/usr/bin/airsaned` is the daemon
    - `/usr/lib/systemd/system/airsaned.service` is the service file
    - `/etc/default/airsane` is the server options
    - `/etc/airsane/access.conf` is the allowed subnets
    - `/etc/airsane/ignore.conf` is the sane backend to ignore
    - `/etc/airsane/options.conf` is the sane backend options
