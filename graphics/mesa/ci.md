Mesa CI
=======

## `gitlab-ci.yml`

- top-level `.gitlab-ci.yml`
  - it includes
    - <https://gitlab.freedesktop.org/freedesktop/ci-templates>
      - `/templates/ci-fairy.yml`
      - `/templates/alpine.yml`
      - `/templates/debian.yml`
      - `/templates/fedora.yml`
    - `.gitlab-ci/image-tags.yml`
    - `.gitlab-ci/lava/lava-gitlab-ci.yml`
    - `.gitlab-ci/container/gitlab-ci.yml`
    - `.gitlab-ci/build/gitlab-ci.yml`
    - `.gitlab-ci/test/gitlab-ci.yml`
    - `.gitlab-ci/farm-rules.yml`
    - `.gitlab-ci/test-source-dep.yml`
    - `docs/gitlab-ci.yml`
    - `src/**/ci/gitlab-ci.yml`
- `.gitlab-ci/container/gitlab-ci.yml` defines jobs for the `container` stage
  - `debian/x86_64_build` builds a x86-64 debian container used for building
    mesa, deqp, etc.
  - `debian/x86_64_pyutils` builds a x86-64 debian container used for running
    python scripts
  - more
  - these jobs extend `.container` which sets `FDO_DISTRIBUTION_EXEC`
    - this is the main script to build the container
    - e.g., `debian/x86_64_test-vk` job will invoke
      `.gitlab-ci/container/debian/x86_64_test-vk.sh`
- `.gitlab-ci/build/gitlab-ci.yml` defines jobs for the `build-for-tests`
  stage
  - `debian-testing` builds mesa from within the `debian/x86_64_build`
    container
  - `python-test` runs and packs python scripts from within the
    `debian/x86_64_pyutils` container
  - more

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

## `panfrost-g610-vk`

- dependencies
  - sanity
    - `sanity` runs `ci-fairy check-merge-request`
  - container
    - `alpine/x86_64_lava_ssh_client` runs `cbuild` to build an alpine
      container for ssh to collabora lava test farm
    - `debian/arm64_build` runs `cbuild` to build a debian container for
      building arm64 images
    - `debian/x86_64_pyutils` runs `cbuild` to build a debian container for
      running python tests
    - `kernel+rootfs_arm64` runs `.gitlab-ci/container/lava_build.sh` inside
      `debian/arm64_build` container
      - this builds a bootable image with deqp inside
  - build-for-tests
    - `debian-arm64` builds mesa inside `debian/arm64_build` container
    - `python-test` runs `.gitlab-ci/run-pytest.sh` and
      `.gitlab-ci/prepare-artifacts-python.sh` inside
      `debian/x86_64_pyutils` container
- extends
  - `.lava-test-deqp` runs `setup-test-env.sh` and `lava-submit.sh` inside
    `debian/x86_64_pyutils` container
  - `.panfrost-test`
  - `.lava-rk3588-rock-5b`
  - `.panfrost-vk-rules`
- `parallel` runs a job multiple times in parallel
    - `CI_NODE_INDEX` and `CI_NODE_TOTAL` are set
- `variables`
  - `DEQP_FRACTION` is fraction of deqp tests to run
  - `FDO_CI_CONCURRENT` is number of deqp-runners per dut
- the gitlab test runner in collabora lava test farm
  - it downloads kernel, dtb, rootfs, etc.
  - it boots dut over network
  - it sets up dut
  - it sshs to dut to run test

## deqp-runner

- `.gitlab-ci/deqp-runner.sh` invokes `deqp-runner`
- `DeqpTomlConfig::test_groups`
  - it reads `--caselists <files>` to get all test cases
  - it calls `TestCommand::split_tests_to_groups` to split test cases into
    test groups
    - `tests` is the vec of all test cases
    - `tests.shuffle()` shuffles the vec with a fixed seed
    - `tests.into_ter().skip(N).step_by(M).filter(F)`
      - `N` is from `--fraction-start`
      - `M` is from `--fraction`
      - `F` is a regex match from `--include-tests`
    - `tests` is splitted into a vec of test groups, where each group has
      `--tests-per-group` test cases
- `parallel_test` runs the test groups in parallel
  - it forks `--jobs` threads and dispatches the test groups to the threads
  - `TestCommand::process_caselist` runs a single test group
    - if a test case is in `--skips`, it is skipped
    - it invokes `deqp-vk` to run the remaining test cases and parses the
      results
      - `TestCommand::translate_result` translates the results
        - it compares with `--baseline` and updates the test case status
        - if a test case fails and matches against `--flakes`, the test case
          status is updated to known flake
    - if there is any unexpected failure, it runs the remaining test cases
      again and marks any status change as flaky
  - `results_collection` collects results from threads
    - `RunnerResults::record_result` records a result
