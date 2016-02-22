Pousblo
=======

## Init

* `psb_gtt_init`
  * `pg->gtt_phys_start` is read from `PSB_PGETBL_CTL` pci config
  * `pg->mmu_gatt_start` is set to `0xE0000000`
  * `pg->gtt_start` and `pg->gtt_pages` are set from GTT BAR
    * 256KB if bar missing.  Can point to 256MB
  * `pg->gatt_start` and `pg->gatt_pages` are set from GATT BAR
    * 128MB if bar missing
  * `dev_priv->gtt_mem` is set to the GATT BAR
  * `pg->stolen_base` is read from `PSB_BSM` pci config (base stolen memory)
    * the physical address of the stolen system memory
    * `pg->gtt_phys_start` is at the top of the stolen memory
  * `pg->stolen_size` is roughly `pg->gtt_phys_start - pg->stolen_base`
  * `pg->vram_stolen_size` is set to `pg->stolen_size`
  * for CPU access,
    * `dev_priv->gtt_map` is mapped from `pg->gtt_phys_start`, non-cached
    * `dev_priv->vram_addr` is mapped from `pg->stolen_base`, write-combined
  * for GPU access,
    * `dev_priv->gtt_map` is set up to map to `pg->stolen_base`
    * the rest entries of `dev_priv->gtt_map` is set up to map to
      `dev_priv->scratch_page`
* memory types
  * the stolen system memory is effectively vram
  * `dev_priv->gtt_mem` is the aperture
    * the first `pg->stolen_size` bytes are reserved for vram
    * the rests can be used for dynamic pages
* `psb_gtt_alloc_range` allocates a resource from `dev_priv->gtt_mem` aperture
* `psb_gtt_pin` pins the pages in GTT so that GPU can use it
  * it calls `psb_gtt_attach_pages` to cache pages (make sure the pages are
    allocated and not in swap?) from the underlying shared memory
  * it calls `psb_gtt_insert` to mark the pages uncached and update
    `dev_priv->gtt_map`
* `psb_gtt_unpin` unpins the pages
  * `psb_gtt_remove` is called to clear `dev_priv->gtt_map` and mark the pages
    write-back
  * `psb_gtt_detach_pages` is called to release the pages (mark it swappable?)

## FB

* `psb_modeset_init`
  * it initializes `dev->mode_config`.  `dev->mode_config.fb_base` is set to the
    base of the stolen memory.  `dev->mode_config.funcs` is set to
    `psb_mode_funcs`
  * it calls `psb_intel_crtc_init` to initialize CRTCs.
    * for each CRTC, a `psb_intel_crtc` is created and added to both
      `dev_priv->plane_to_crtc_mapping` and `dev_priv->pipe_to_crtc_mapping`
    * `drm_crtc_init` also adds the crtc to `dev->mode_config.crtc_list`
    * `psb_intel_crtc_funcs` is used as the crtc funcs
    * `mrst_helper_funcs` is used as the crtc helper funcs
  * finally, `psb_setup_outputs` is called.
    * two connector properties, `dev->mode_config.scaling_mode_property` and
      `dev_priv->backlight_property`, are created
    * `dev_priv->ops->output_init`, i.e. `mrst_output_init`, is called first.
    * `mrst_lvds_init` creates a `psb_intel_output`.  It holds a `drm_connector`
      and a `drm_encoder`.
      * the connector is initialized with `psb_intel_lvds_connector_funcs` and
        `psb_intel_lvds_connector_helper_funcs`
      * the encoder is initialized with `psb_intel_lvds_enc_funcs` and
        `mrst_lvds_helper_funcs`
      * the encoder is attached to the connector
      * two optional connector properties are attached to the connector
      * there are no EDID data so `mrst_lvds_get_configuration_mode` is called
        to determine `mode_dev->panel_fixed_mode`
    * for each connector, one if no hdmi, `possible_crtcs` and `possible_clones`
      of the attached encoder are updated.
* `psb_fbdev_init` creates and initializes `dev_priv->fbdev`
  * `psb_fb_helper_funcs` is the helper funcs
  * `drm_fb_helper_initial_config` will parse `video=...`.  It calls
    `connector->funcs->fill_modes` on all fb connectors.
    * finally, `psbfb_create` is called.

