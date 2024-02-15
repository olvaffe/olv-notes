WireGuard
=========

## Overview

- wireguard works by adding a network interface
  - `ip link add wg0 type wireguard`
  - `wg0` can be configured like any regular network interface
  - `wg` can be used for wireguard-specific configuration
- a wireguard interface must have a private key
  - `wg genkey > ./private`
  - `wg set wg0 private-key ./private`
  - the interface derives the public key automatically
  - the interface also has a listening udp port which is 51820 by default
- a wireguard interface also has a list of peers (remote wireguard interfaces)
  - `wg set wg0 peer <public-key-of-peer> allowed-ips <allowed-subnet> endpoint
    <public-ip-and-udp-port-of-peer>`
- or, more commonly, write the configuration to a config file and
  - `wg setconf wg0 ./config.conf`
- cryptokey routing
  - when a wireguard interface sends an ip packet,
    - it finds the peer based on `allowed-ips` of the peer and the dst addr of
      the ip packet
    - it encrypts the ip packet with the public key of the peer
    - it wraps the encrypted ip packet in a udp packet
    - it sends the udp packet to the current endpoint of the peer
  - when a wireguard interface receives a udp packet,
    - it decrypts the encrypted ip packet with the private key
    - it finds the peer based on the src addr of the udp packet
      - if the src addr is unknown, or if the public key of the peer cannot
        authenticate the ip packet, it tries each peer's public key until the
        first peer that can authenticate the ip packet
      - it then remembers the src addr as the current endpoint of the peer
    - it checks if the src addr of the ip packet is on `allowed-ips` of the
      peer to accept the ip packet

## Protocol

- <https://www.wireguard.com/protocol/>
- the private key is generated using Curve25519
- the public key is generated from the private key
- first message: initiator to responder
  - `message_type` is 1
  - `unencrypted_ephemeral` is the initiator's public key
  - `encrypted_static` is ChaCha20Poly1305 of various stuff
  - `encrypted_timestamp` is ChaCha20Poly1305 of various stuff
  - `mac1` is Keyed-Blake2s of various stuff
  - `mac2` is Keyed-Blake2s of various stuff
  - when the responder receives the message, it does all the operations
    above in reverse to derive the initiator's state
- second message: responder to initiator
  - `message_type` is 2
  - `unencrypted_ephemeral` is the responder's public key
  - `hash` is BLAKE2s hash of the initiator's endpoint, the responder's
    public key, and various stuff
  - `encrypted_nothing` is ChaCha20Poly1305 of various stuff
  - `mac1` is Keyed-Blake2s of various stuff
  - `mac2` is Keyed-Blake2s of various stuff
  - when the initiator receives the message, it does all the operations
    above in reverse to derive the responder's state
- after the two messages have been exchanged, keys can be calculated
- subsequent messages: exchange of data packets
  - `message_type` is 4
  - `counter` is a sequence number
  - `encrypted_encapsulated_packet` is ChaCha20Poly1305 of the ip packet
