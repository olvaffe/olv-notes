Linux net drivers
=================

## soc ethernet driver

- terms
  - phy implements layer 1
  - mac implements layer 2
    - gmac is an impl of mac
  - they are interconnected by mii
    - mdio is an impl of mii
- e.g., `rk3588-base.dtsi` has
  - `gmac1: ethernet@fe1c0000`
    - `compatible = "rockchip,rk3588-gmac", "snps,dwmac-4.20a";`
    - `mdio1: mdio`
      - `compatible = "snps,dwmac-mdio";`
  - board dts, such as `rk3588s-orangepi-5.dts`, adds a child node compatible
    with `ethernet-phy-ieee802.3-c22`
- `rk_gmac_probe` probes gmacs
  - `stmmac_probe_config_dt` calls `stmmac_mdio_setup` which calls
    `stmmac_of_get_mdio` to find the mdio node
  - `stmmac_dvr_probe` calls `stmmac_mdio_register` which calls
    `of_mdiobus_register` to register the mdio node
  - `__of_mdiobus_register` calls `__of_mdiobus_parse_phys` which calls
    `of_mdiobus_register_phy` to register the `ethernet-phy-ieee802.3-c22`
    phy child node
  - `fwnode_mdiobus_phy_device_register` calls `phy_device_register` to
    register a `phy_device`
    - `phy_id` is queried via `get_phy_c22_id`
- `module_phy_driver(motorcomm_phy_drvs)` registers several `phy_driver`s
  - when a `phy_device` has id `PHY_ID_YT8531`, `yt8531_probe` probes the phy
    device

## veth

- `ip link add ve-host type veth peer name ve-guest`
  - it is handled by `veth_link_ops`
  - `veth_setup` sets up a new `net_device`
  - `veth_newlink` initilizes the new link
    - it creates the peer first
      - `rtnl_create_link` create the peer, also with `veth_link_ops`
      - `register_netdevice` registers the peer
      - `rtnl_configure_link` configures the peer
    - `register_netdevice` registers the dev itself
    - the dev and the peer points to each other in their `peer` member
    - `veth_init_queues` is called on both the dev and the peer
- `veth_xmit` is called by `dev_hard_start_xmit` when there is an outgoing
  frame
  - `veth_forward_skb` calls `__netif_rx` such that the frame is queued for
    receiving
- `veth_poll` is called by `napi_poll` when there is an incoming frame
  - `veth_xdp_rcv` calls `napi_gro_receive` or `netif_receive_skb` to move the
    skb to the upper layer
- in other word, a frame sent on one end is received on another, just like two
  connected devices

## macvlan

- `ip link add mv-guest link eth0 type macvlan mode bridge`
  - it is handled by `macvlan_link_ops`
  - `macvlan_setup` sets up a new `net_device`
  - `macvlan_newlink` initilizes the new link
    - `vlan` is the link itself
    - `macvlan_port_create` calls `netdev_rx_handler_register`
      - the port is assoicated with the lower device and is shared by all
        macvlans of the lower device
      - it calls `netdev_rx_handler_register` to intercept all incoming frames
        of the lower device by `macvlan_handle_frame`
    - `vlan->lowerdev` is `eth0`
    - `vlan->mode` is `MACVLAN_MODE_BRIDGE`
- `macvlan_start_xmit` is called by `dev_hard_start_xmit` when there is an
  outgoing frame
  - if the dst is another macvlan, `dev_forward_skb` calls `netif_rx_internal`
    to add the skb to the lower device
  - otherwise, `skb->dev` is set to the lower device and
    `dev_queue_xmit_accel` is called to transmit the skb from the lower device
- `macvlan_handle_frame` is called by `__netif_receive_skb_core` when the
  lower device receives an incoming frame
  - `macvlan_forward_source` is for `MACVLAN_MODE_SOURCE` and does nothing
  - if the mac dst is not a macvlan, `RX_HANDLER_PASS` is returned
    - `__netif_receive_skb_core` will continue to handle the frame normally
      using the lower device
  - otherwise, `skb->dev` is set to the macvlan and `RX_HANDLER_ANOTHER` is
    returned
    - `__netif_receive_skb_core` will restart and will handle the frame using
      the macvlan
- in other word,
  - a frame sent by one macvlan is
    - received by another macvlan, if the mac dst is the other macvlan, or
    - sent by the lower device
  - a frame received by the lower device is
    - handled normally, or
    - handled as if it is received by the macvlan
