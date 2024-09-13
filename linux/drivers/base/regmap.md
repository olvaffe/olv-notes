Linux regmap
============

## regmap

- `devm_regmap_init` allocs and inits a `regmap`, and calls
  `regmap_attach_dev` to add the regmap as a `devres`
  - `regmap_bus` and `regmap_config` define valid regs and how to access them
- `devm_regmap_add_irq_chip` allocs and inits a `regmap_irq_chip_data`, and
  adds it as a devres
  - it calls `request_threaded_irq` to register `regmap_irq_thread` to handle
    the irq
  - it uses the associated regmap to enable/ack/disable irqs
- `regmap_read`
  - `map->lock` to lock the regmap
  - `map->reg_read` to lock the regmap
  - `map->unlock` to unlock the regmap
