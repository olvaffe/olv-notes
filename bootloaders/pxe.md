PXE
===

## Server

- PXE relies on DHCP (with PXE-specific vendor info) and TFTP
- example
  - dnsmasq -d --listen-address 192.168.0.1 \
    --port 0 \
    --dhcp-range 192.168.0.10,192.168.0.20 --log-dhcp \
    --pxe-service=0,"Raspberry Pi Boot" \
    --enable-tftp --tftp-root <path>
  - make sure `<path>` is accessible by nobody or specify `--user`

