Mesa CI
=======

## `vkcts-vega10-valve`

- `vkcts-vega10-valve`
  - it extends
    - `.vkcts-test-valve`
      - it sets `DEQP_VER: vk` and `RADV_PERFTEST: nv_ms,gpl`
      - `.b2c-test-radv-vk`
        - `.radv-valve-rules` creates the job when core, vk, or radv files are
          changed
        - `.test-radv`
          - it sets `VK_DRIVER: radeon`, `DRIVER_NAME: radv`,
            `MESA_SPIRV_LOG_LEVEL: error`, `ACO_DEBUG: validateir,validatera`,
            `MESA_VK_IGNORE_CONFORMANCE_WARNING: 1`, `radv_require_etc2: 'true'`
        - `.b2c-test-vk`
          - it extends
            - `.use-debian/x86_test-vk`
            - `.b2c-test`
          - it needs
            - `debian/x86_test-vk`
            - `debian-testing`
      - `.deqp-test-valve`
    - `.vega10-test-valve`
      - it sets `FDO_CI_CONCURRENT: 16`
      - it adds tag `amdgpu:codename:VEGA10`
    - `.radv-valve-manual-rules`
      - `.valve-farm-rules` requires `VALVE_FARM` to be `online`
      - `.vulkan-manual-rules` creates the job when core or vk files are
        changed
  - it sets `GPU_VERSION: radv-vega10-aco`
- `debian/x86_build` container job runs `debian/x86_build.sh`
  - it builds all build dependencies
- `debian-testing` build job runs two scripts
  - `CI_PRE_CLONE_SCRIPT` is executed and mesa is automatically cloned
  - `.gitlab-ci/meson/build.sh` builds mesa
  - `.gitlab-ci/prepare-artifacts.sh` packages the artifacts and uploads them
- I guess the entire job is sent to the gitlab runner on valve farm gateway
  - the job runs inside `debian/x86_test-vk` container
