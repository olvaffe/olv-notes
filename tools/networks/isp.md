# ISP

## Cable

- DOCSIS, Data Over Cable Service Interface Specification
  - DOCSIS 1.0
    - 1997
    - up to 40Mbit/s downstream and 10Mbit/s upstream
    - evolved from proprietary cable modems
  - DOCSIS 1.1
    - 2001
    - added voip and QoS
  - DOCSIS 2.0
    - 2001
    - up to 40Mbit/s downstream and 30Mbit/s upstream
  - DOCSIS 3.0
    - 2006
    - up to 1Gbit/s downstream and 200Mbit/s upstream
    - IPv6 support
  - DOCSIS 3.1
    - 2013
    - up to 10Gbit/s downstream and 1Gbit/s upstream
  - DOCSIS 4.0
    - 2017
    - up to 10Gbit/s downstream and 6Gbit/s upstream
- CMTS, Cable Modem Termination System
  - A CMTS has N downstream ports and 4 to 6 times for upstream ports
    - each downstream port is usually connected to 4 to 6 neighborhoods while
      each upstream port is connected to one neighborhood, because the return
      path is more noisy
  - A CMTS has full control over the modem's configuration
    - it can reboot the modem
    - it can push fw update to the modem
  - The traffic is encrypted such that two customers cannot listen to each
    other's traffic
    - it also allows the operator to refuse service to uncertified modems or
      unauthorized customers
- Connection Handshake
  - each modem has
    - a 12-char mac registered with isp
    - an x509 certificate signed by the vendor
  - cmts validates both the mac and the signed x509 certificate
    - as such, mac spoofing is not possible
  - modem uses dhcp to get a temp ip and uses tftp to download its config
  - modem applies the config, is assigned an SID (service id), and exchanges
    the encryption key with cmts
