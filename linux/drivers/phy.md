Kernel PHY
==========

## PHY driver

- `devm_phy_create` creates a phy
- `devm_of_phy_provider_register` registers a of phy provider
  - `__of_phy_provider_register` allocs a `phy_provider` and adds it to
    `phy_provider_list`

## Consumer

- the consumer dt node often has something like
  - `phys = <&u2phy2_host>;`
  - `phy-names = "usb";`
- when the consumer driver calls `devm_of_phy_get`,
  - `of_phy_provider_lookup` looks up `phy_provider` in `phy_provider_list`
    using the dt props
