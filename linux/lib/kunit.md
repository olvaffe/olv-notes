Kernel KUnit
============

## Arch

- <https://docs.kernel.org/dev-tools/kunit/architecture.html>
- `kunit_test_suite` defines a test suite in the `.kunit_test_suites` section
- `vmlinux.lds.h` defines `__kunit_suites_start` and `__kunit_suites_end` for
  the test suite array
- `kunit_run_all_tests` runs the tests before `kernel_init_freeable` returns
- similarly, when a module is loaded
  - `find_module_sections` parses `.kunit_test_suites` section for test suites
  - `kunit_module_init` runs the tests

## `kunit.py`

- `tools/testing/kunit/kunit.py config` configs kunit
  - `tools/testing/kunit/configs/default.config` is copied to
    `.kunit/.kunitconfig` and `.kunit/.config`
  - `make ARCH=um O=.kunit olddefconfig` to config for UML
  - to customize,
    - `--kunitconfig` specifies an alternative defconfig
    - or, edit `.kunit/.kunitconfig` directly
- `./tools/testing/kunit/kunit.py build` buils the kernel
  - `make ARCH=um O=.kunit --jobs=<N>`
- `./tools/testing/kunit/kunit.py exec` executes the tests
