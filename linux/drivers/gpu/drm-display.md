## DRM Drivers

- an eDP controller is similar to an any-to-eDP bridge
  - `drm_dp_aux_init` inits a DP aux channel
  - `devm_of_dp_aux_populate_bus` adds the downstream device
  - after a dp-aux driver binds to the device, `done_probing` is called
    - `devm_drm_of_get_bridge` finds the device
    - `devm_drm_bridge_add` adds self as a bridge
  - when the pixel source driver calls `drm_bridge_attach`, `attach` is called
    - `drm_dp_aux_register` registers the DP aux channel
    - `drm_bridge_attach` attaches the downstream device 
  - when the pixel souce driver calls `drm_bridge_connector_init`, it inits a
    connector for the entire bridge chain
  - later, when `drm_atomic_helper_commit_tail` commits,
    `drm_atomic_bridge_chain_enable` enables all bridges
