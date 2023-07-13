Digital Rights Management
=========================

## Technologies

- <https://en.wikipedia.org/wiki/Digital_rights_management#Technologies>
- Verification
  - Product keys: users must enter product keys to continue
    - circumvention: patch the software to bypass product key verification or
      RE the verification algorithm and generate valid keys
  - Activation limits: only N devices can access the contents
  - Persistent online (always-on) DRM : during use, periodically check against
    a server
    - circumvention: RE the protocol and redirect the traffic to a custom
      server
- Encryption: the content must be decrypted before consumption
- Copy restriction: disallow copying, printing, forwarding
- Runtime restrictions: refuse to access the content unless the software is
  signed
- Regional lockout: disallow unless in a specific region, using ip addresses,
  etc.
- Tracking
  - digital watermarks embedded in content
  - metadata
- Hardware: require a specific hardware to access content

## Implementations

- multimedia
  - google widevine
  - apple fairplay
  - microsoft playready
- ebook
  - amazon
  - adobe access
  - marlin

## DRM in Video Streaming

- content
  - the streaming protocol
    - MPEG-DASH, HLS (apple), SmoothStreaming (microsoft), HTTP Dynamic
      Streaming (adobe), etc.
  - the media container
    - CMAF, MP4, MPEG-TS, etc.
  - the encryption layer
    - widevine, fairplay, playready, etc.
  - the codec
    - avc, hevc, av1, etc.
- streaming protocols
  - Apple HLS (HTTP Live Streaming)
    - 2009
    - `.m3u8` as the manifest
    - MPEG-TS as the container
    - CBC as the encryption
    - AVC as the codec
  - MPEG-DASH
    - 2012
    - `.mpd` as the manifest
    - fragmented MP4 as the container
    - any encryption
    - any codec
- container formats
  - the divergence on container formats is annoying
    - fMP4 is superior to MPEG-TS
    - but apple and legacy devices only support MPEG-TS
    - content providers need to package both which adds costs
  - CMAF is gaining traction
    - both HLS and MPEG-DASH can support CMAF now
- encryptions
  - there are four common encryption schemes
    - `cenc` for AES-CTR
    - `cbc1` for AES-CBC
    - `cens` for AES-CTR subsample pattern
    - `cbcs` for AES-CBC subsample pattern
  - fairplay uses `cbcs`
  - widevine and playready used to use `cenc`
    - they support `cbcs` now
- content playback in browsers
  - the player decrypts, decodes, and displays the content
  - it downloads `.mpd` from CDN first
  - it parses `.mpd` and passes init data to CDM (Content Decryption Module)
    - CDM is a signed closed-source browser plugin provided by the DRM
      provider
  - CDM generates a license request
  - the player sends the license request to the license server and receives
    the decryption key
  - the player forwards the decryption key and the content to CDM
  - depending on the security level,
    - high: CDM is in TEE; it decrypts, decodes, and displays the content
      - streaming usually requires this for 1080p+
    - medium: CDM is in TEE; it decrypts the content; player decodes and displays
    - low: CDM is NOT in TEE
      - streaming usually limits to 720p
- netflix computer requirements for html5 player
  - <https://help.netflix.com/en/node/23742>
  - chrome: up to 4K on macos and cros; up to 720p on win/lin
  - firefox: up to 720p
  - opera: up to 720p
  - safari: up to 4K on macos 11+
  - edge: up to 4K

## Google Widevine

- CDM runs on cpu
- another component, OEMCrypto, runs in TEE

## W3C EME

- <https://wiki.mozilla.org/Media/EME>
  - Encrypted Media Extensions
  - server provides decryption keys and content
  - client (player) forward the messages to CDM to decrypt, decode, and
    display the video
  - EME defines a javascript api for clients to do that
