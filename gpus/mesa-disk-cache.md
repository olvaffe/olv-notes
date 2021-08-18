Mesa Disk Cache
===============

## In-Memory Cache

- apps should use a `VkPipelineCache`
  - if they don't, the driver may transparently create and use one
  - this is because different `VkGraphicsPipelineCreateInfo`s may hash and
    compile to the same shader
  - the driver does not want to waste the time
- `VkPipelineCache` is an in-memory cache
  - `sha1->pipeline` lookup
  - with reference counting, can cache pointers to pipelines than just binary
    code

## Disk Cache

- 
