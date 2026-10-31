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
    - CTAP2 is deisgned for both 2FA and passwordless (resident passkey)
  - an authenticator supporting either (or both) protocol is referred to as a
    U2F or FIDO2/WebAuthn authenticator
    - a security key is an example of an authenticator
  - 2FA is non-resident
    - registration
      - authenticator generates a key pair for each relying party (domain)
      - authenticator exports the key pair with privkey encrypted
        - decrypted privkey never leaves authenticator
        - encrypted privkey is called a credential id
      - relying party stores the key pair
    - authentication
      - relying party asks for username and password
      - relying party looks up and returns the encrypted key pair
      - authenticator decrypts the key pair to solve the challenge
        - authenticator requires UP, user presense, to decrypt
  - passwordless is resident
    - registration
      - authenticator stores the key pair as well as account metadata
        - there is a limit on number of resident keys
    - authentication
      - relying party sends rp id (domain name) and challenge
        - no username/password needed
      - authenticator looks up the resident key, solves the challenge, and
        sends everything including account metadata back to rp
        - authenticator requires UV, user verification, to look up
- C WebAuthn involves a website, a browser, and an authenticator
  - the website conforms to WebAuthn Relying Party
  - the browser conforms to WebAuthn Client
  - the authenticator conforms to CTAP1/UAF or CTAP2

## Open Source

- <https://github.com/lowrisc/opentitan>
  - root of trust hw designs
  - early grey is for discrete rot
  - darjeeling is for integrated rot
- <https://github.com/google/opensk>
  - fido2 fw impl

## Command Line

- `fido2-token -L` lists
  - `-L` alone lists authenticators
  - `-L -r <dev>` lists relaying parties
    - `<rp_idx>: <rp_hash> <rp_id>`
      - hash is `echo -n <rp_id> | openssl dgst -sha256 -binary | base64`
  - `-L -k <rp_id> <dev>` lists resident creds for the relaying party
    - `<cred_idx>: <cred_id> <user.name> <user.id> <algo> <flags> <payload>`
- `fido2-token -I` shows info
  - `-I` alone shows protocol version and caps
  - `-I -k <rp_id> -i <cred_id> <dev>` shows pubkey of the resident cred
- `fido2-token -S` sets
  - `-S` alone sets pin after factory reset
- `fido2-token -G` gets
- `fido2-token -D` deletes/disables
- `fido2-token -C` changes the existing pin
- `fido2-token -R` factory resets

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

## PC/SC Smartcards

- history
  - MS defines PC/SC spec, and provides `winscard.dll` to clients
  - pcsc-lite is a partial impl of the spec, and provides `libpcsclite.so`
- pcsc-lite
  - client uses `libpcsclite.so` to talk to `pcscd` over unix socket
  - `pcscd` loads an IFD, aka a driver plugin, to talk to the hardware
  - the most common IFD is `libccid.so`, which supports USB CCID and ICCD
    - usb card readers often support CCID
    - usb keys often support ICCD
  - for EZ100PU, there is <https://github.com/drinkcat/ezIFD>
- opensc
  - `app -> PKCS #11 (opensc impl) -> pcsc-lite`
    - PKCS #11 defines a standard C API
- taiwan
  - MOICA card: `browser -> HiPKILocalSignServerApp -> PKCS #11 (HiCOS impl) -> pcsc-lite`
    - `HiPKILocalSignServerApp` provides a web service on port 61161
    - the api is json-based
  - NHI card: `browser -> mLNHIICC -> pcsc-lite`
    - `mLNHIICC` provides a web service on port 7777
    - the api is also json-based
