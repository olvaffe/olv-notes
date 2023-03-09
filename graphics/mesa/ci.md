Mesa CI
=======

## `vkcts-vega10-valve`

- it has two chains of dependencies
  - first
    - `debian/x86_build-base`
    - `debian/x86_build`
    - `debian-testing`
  - second
    - `debian/x86_test-base`
    - `debian/x86_test-vk`
- container job: `debian/x86_build-base`
  - this builds a generic debian x86 base image with `DEBIAN_BASE_TAG` tag
  - the image is used to build mesa and stuff
  - it extends fdo rules and sets a few variables
  - it uses docker image registry and normally does not need to build anything
    - otherwise, it executes `.gitlab-ci/debian/x86_build-base.sh`?
- container job: `debian/x86_build`
  - this builds a generic debian x86 base image with `DEBIAN_BUILD_TAG` tag
  - it also completes immediately due to image registry
    - otherwise, it executes `.gitlab-ci/debian/x86_build.sh`?
- build job: `debian-testing`
  - this uses `debian/x86_build` to build mesa and stuff
  - it executes `CI_PRE_CLONE_SCRIPT` which exits early because mesa is already cloned to
    `CI_PROJECT_DIR` (`/builds/mesa/mesa`)
  - it executes `.gitlab-ci/meson/build.sh` to build mesa
  - it executes `.gitlab-ci/prepare-artifacts.sh` to prepare artifacts
  - it uploads the artifacts
- container job: `debian/x86_test-base`
  - this builds a generic debian x86 base image with `DEBIAN_BASE_TAG` tag
  - it sets `KERNEL_URL` variable such that all DUTs boot the same kernel
  - it completes immediately because of image registry
    - otherwise, it executes `.gitlab-ci/debian/x86_test-base.sh`?
- container job: `debian/x86_test-vk`
  - this builds a generic debian x86 base image with `DEBIAN_X86_TEST_VK_TAG`
  - it completes immediately because of image registry
    - otherwise, it executes `.gitlab-ci/debian/x86_test-vk.sh`?
- amd job: `vkcts-vega10-valve`
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
- `.b2c-test`
  - `before_script`
    - it creates `JOB_FOLDER`
    - it executes `generate-env.sh` to generate `set-job-env-vars.sh`
    - it copies `setup-test-env.sh`
    - it copies `INSTALL_TARBALL`
    - `B2C_TEST_SCRIPT`, which is executed after the container is started on
      DUT, is set to
      - `source ./set-job-env-vars.sh`
      - `source ./setup-test-env.sh`
      - `tar xf ${INSTALL_TARBALL_NAME}`
      - `./install/deqp-runner.sh`
  - `script`
    - it executes `executorctl` to dispatch the job to DUT
    - the DUT is netbooted, starts the container, and `B2C_TEST_SCRIPT`
      - <https://mupuf.pages.freedesktop.org/valve-infra/>
    - `deqp-runner run
         --deqp /deqp/external/vulkancts/modules/vulkan/deqp-vk
         --output /builds/mesa/mesa/results
         --caselist /tmp/case-list.txt
         --skips /builds/mesa/mesa/install/all-skips.txt /builds/mesa/mesa/install/radv-skips.txt /builds/mesa/mesa/install/x11-skips.txt
         --flakes /builds/mesa/mesa/install/radv-vega10-aco-flakes.txt
         --testlog-to-xml /deqp/executor/testlog-to-xml
         --jobs 16
         --baseline /builds/mesa/mesa/install/radv-vega10-aco-fails.txt
         --
         --deqp-surface-width=256
         --deqp-surface-height=256
         --deqp-surface-type=pbuffer
         --deqp-gl-config-name=rgba8888d24s8ms0
         --deqp-visibility=hidden`
