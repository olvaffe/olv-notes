Chromium crossbench
===================

## Usage

- <https://chromium.googlesource.com/crossbench> is the framework
- <https://chromium.googlesource.com/chromium/web-tests/> is the tests
- fetch
  - `mkdir src; cd src`
  - `fetch web-tests`
- deps
  - `cd cuj/crossbench/runner`
  - `pip install poetry`
  - `poetry env use 3.11`
  - `PYTHON_KEYRING_BACKEND=keyring.backends.null.Keyring poetry install`
- run
  - `poetry run python run.py --platform adb --device <serial> --tests memory-pressure --variants 8g`
- internal
  - starts `com.android.chrome` on dut
  - starts `perfetto` on dut
  - sets up adb port forwarding to `chrome_devtools_remote` on dut
  - starts webdriver on host to control chrome
    - `crossbench -> webdriver -> chrome`

## CUJ

- `memory-pressure`
  - `probe-config.hjson` enables `screenshot`, `dump_html`, `trace_processor`,
    `perfetto`, `downloads`, `meminfo` probes
    - `trace_processor` parses trace colleced by `perfetto` to generate
      metrics
  - `browser-flags.hjson` specifies chrome flags
  - `<variant>.page-config.hjson` opens 100 tabs
    - each tab runs `action-blocks/scripts/alloc.js` to allocate cpu memory
      and waits for 2s
    - the allocation size is `MEMORY_GB / 50`, causing overcommit beyond 50 tabs
  - metrics
    - `cuj.memory-pressure.page_load_duration` is the page load time in ms of each tab
    - `cuj.memory-pressure.allocation_duration` is the mem alloc time in ms of each tab
    - `cuj.memory-pressure.tabs_alive` is the total alive tabs after each new tab
    - `cuj.memory-pressure.average_tabs_alive_after_first_kill` is the average alive tables after first kill
    - `cuj.memory-pressure.tabs_alive_after_first_kill` is the total alive tabs after first kill
