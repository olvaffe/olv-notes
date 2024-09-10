Linux Interconnect
==================

## Controllers

- a soc typically has multiple interconnects
- an interconnect controller
  - takes inputs from hw blocks and sw
  - controls the throughput, latency, priority, etc. of hw blocks
  - the goal is to save power or maximize performance
- terms
  - a provider is a driver for a interconnect controller
  - a node is a port of the interconnect controller
  - a path is a seqeuence of nodes, with the first/last nodes called endpoints
  - a consumer is an entity that makes use of paths exported by a provider
- controller driver probe
  - `devm_kzalloc` to alloc a `icc_provider`
  - `icc_node_create` to create a `icc_node` for each port
  - `icc_node_add` to add nodes to the provider
  - `icc_provider_register` to register the provider

## Consumers

- `devm_of_icc_get` returns an `icc_path`
  - it calls `of_icc_get_from_provider` to get the src node and the dst node
  - a path is created and initialized
- `icc_set_bw` sets the bandwidth of a path
  - this expresses the need of the consumer
  - the provider aggregates reqs from all consumers and picks the ideal bw

## MediaTek

- `CONFIG_INTERCONNECT_MTK_DVFSRC_EMI` enables support for the EMI
  interconnect
  - EMI connects to DRAM, CPU, GPU, and MMSYS
    - MMSYS in turn connects to VPU, DISP, VDEC, VENC, IMG, and CAM
  - EMI does not make decisions nor controls the hw blocks
  - instead, it calls `mtk_dvfsrc_send_request` to forward aggregated requests
    to DVFSRC
    - DVFSRC driver is provided by `CONFIG_MTK_DVFSRC` under
      `drivers/soc/mediatek`

