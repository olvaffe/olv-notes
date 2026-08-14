# FIDO

## Overview

- FIDO Alliance
  - "Fast IDentity Online"
  - founded in 2013
- Standards
  - U2F 1.0, 2014
  - UAF 1.0, 2014
  - FIDO 2.0, 2015
    - based on U2F 1.0
    - submitted to W3C
  - UAF 1.1, 2017
  - U2F 1.2, 2017
  - CTAP 2.0, 2017
    - based on U2F 1.2
  - UAF 1.2, 2017
  - W3C WebAuthn, 2019
    - based on FIDO 2.0
    - collaboration between FIDO alliance and W3C
- CTAP stands for Client to Authenticator Protocol
  - it defines two protocols: CTAP1/U2F and CTAP2
    - CTAP1/U2F is designed for 2FA
    - CTAP2 is deisgned for both 2FA and passwordless
  - an authenticator supporting either (or both) protocol is referred to as a
    U2F or FIDO2/WebAuthn authenticator
    - a security key is an example of an authenticator
- W3C WebAuthn involves a website, a browser, and an authenticator
  - the website conforms to WebAuthn Relying Party
  - the browser conforms to WebAuthn Client
  - the authenticator conforms to CTAP1/UAF or CTAP2

## Command Line

- `fido2-token -L [<dev>]` lists various stuff
  - `-L` lists authenticators
- `fido2-token -I <dev>` shows various stuff
  - `-I` shows protocol version and caps
- `fido2-token -G <dev>` gets various stuff
- `fido2-token -S <dev>` sets various stuff
  - `-S` sets pin
- `fido2-token -D <dev>` deletes/disables various stuff
- `fido2-token -C <dev>` changes various stuff
  - `-C` changes pin
- `fido2-token -R <dev>` resets device

## Yubikey

- <https://wiki.archlinux.org/title/YubiKey>
- depending on the model, it can
  - act as a FIDO2/WebAuthn authenticator using the CTAP2 protocol
  - act as a U2F authenticator using the CTAP1/U2F protocol
  - act as a smartcard using the CCID protocol
    - this allows storage of PGP and PIV secret keys, as well as OATH
      credentials
  - act as a keyboard using the HID protocol
    - generate OTP passwords
    - type a static password
    - handle challenge-response requests

## Enterprise

- corp sso cookie
  - user authenticates to the corp login service to get the corp sso cookie
  - when using the browser, the sso cookie is stored in browser cookie jar
  - when using cli tools, the sso cookie is stored somewhere on disk
- corp network
  - user uses sso cookie to access (virtual) corp network service to gain internal
    network access
  - user gains limited internal network access if the machine lacks machine
    cert
    - user uses sso cookie to access internal CA service to refresh machine cert
      - machine cert is typically stored in tpm
    - user gains full internal network access with the refreshed machine cert
  - for physical corp network, machine cert is used with 802.1X to gain access
- internal web service
  - user uses sso cookie to access internal web service
  - if the internal web service uses google account credential, user can be
    redirected to google login
- internal non-web service
  - user uses sso cookie to access internal non-web service
    - it works the same way except a cli tool is used
  - if the internal non-web service uses corp account credential, user can be
    rejected
    - user needs to use sso cookie to access internal CA service to refresh
      user cert
      - user cert is typically stored on disk
    - for ssh, user needs to use sso cookie to access internal CA service to
      refresh ssh cert
      - ssh cert is typically stored on security key
