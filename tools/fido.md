FIDO
====

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
