Kernel DMA Engine
=================

## DMA Controller

- take pl330 on arm for exmple
  - `compatible = "arm,pl330", "arm,primecell";`
- when `of_platform_default_populate` is called during boot, if a node is
  compatible with `arm,primecell`, `of_platform_bus_create` calls
  `of_amba_device_create` to add it as a `amba_device`
- `pl330_probe` probes the amba device
  - `dma_async_device_register` registers a `dma_device`
  - `of_dma_controller_register` registers a `of_dma` to `of_dma_list`, which
    consumers will use to get their `dma_chan`

## Consumer

- a consumer of node usually has something like
  - `dmas = <&dmac0 10>, <&dmac0 11>;`
  - `dma-names = "tx", "rx";`
- the consumer driver calls `dma_request_chan` to request a `dma_chan`
  - this calls into `of_dma_request_slave_channel`
  - it loops through `dma-names` to find a match
  - `of_dma_find_controller` lops through `of_dma_list` to find the `of_dma`
    registered by the controller driver
  - `ofdma->of_dma_xlate` returns the `dma_chan`
