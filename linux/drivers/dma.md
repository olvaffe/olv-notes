Kernel DMA Engine
=================

## DMA Engine

- some systems have devices that share an external dma engine
  - some devices have their own built-in dma engines insteadd
  - external dma engines are under `drivers/dma`
  - not to be confused with dma mapping, which is under `kernel/dma`
    - a page must be dma-mapped to be dma-able by built-in or external engines
- a (external) dma engine has several channels and several request lines
  - a request line is a pysical connection from a device to the engine
  - a channel is what does the transfers
- A dma engine is represented by a `dma_device` and is registered with
  `dma_async_device_register`
  - `dma_cap_set` sets what the engine is capable of
    - `DMA_MEMCPY`, memory-to-memory copies
    - `DMA_SLAVE`, device-to-memory copies
- Using a dma engine
  - allocate a dma slave channel
    - `dma_request_chan`
  - config the channel
    - `dmaengine_slave_config`
  - get the descriptor for a transaction
    - `dmaengine_prep_slave_sg` transfers sg list to/from device
    - `dmaengine_prep_dma_cyclic` transfers ring buffer to/from device
    - direction, `dma_transfer_direction`, is indicated
      - `DMA_MEM_TO_MEM`
      - `DMA_MEM_TO_DEV`
      - `DMA_DEV_TO_MEM`
      - `DMA_DEV_TO_DEV`
  - submit the transaction to the pending queue
    - `dmaengine_submit`
  - issue transfers
    - `dma_async_issue_pending`

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
