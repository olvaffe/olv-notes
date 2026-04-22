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
- `tab-stress`
  - `<variant>.page-config.hjson` opens 100 tabs and cycles through them
    - the variant determines urls to open
      - `blank` opens `about:blank`
      - `heavy` opens `https://html.spec.whatwg.org/`
      - `real-tab` opens real-world urls
      - `multi-process` opens `https://chromium-workloads.web.app/web-tests/main/cuj/crossbench/cujs/tab-stress/multiprocess/frame-container/?count=5`
    - cycling switches to the next tab one-by-one
  - metrics
    - `cuj.tab-stress.total_renderers` is the number of alive renderer processes

## MotionMark

- <https://github.com/WebKit/MotionMark>
  - `python -m http.server -d MotionMark` and open
    <http://localhost:8000/index.html>
- `window.addEventListener("load", ...)`
  - `window.sectionsManager` is `SectionsManager`
  - `window.benchmarkController` is `BenchmarkController`
  - `BenchmarkController::initialize` detects the framerate
    - it disables the start button
    - `detectFrameRate`
      - `requestAnimationFrame` calls `tick` on next repaint
      - it measures the time needed for 300 repaints
      - `finish` returns the closest well-known framerate, typically 60
    - `frameRateDeterminationComplete` shows the detected framerate and
      enables the start button
- `<button id="start-button" onclick="benchmarkController.startBenchmark()">`
  calls `BenchmarkController::startBenchmark`
  - `determineCanvasSize` adds a class to `<body>` depending on the screen
    width
    - `window.matchMedia` queries the screen width
      - <https://en.wikipedia.org/wiki/Media_queries> 
    - `document.body.classList.add` adds the class
    - CSS picks the size for `frame-container`
      - `small` is 568x320
      - `medium` is 900x600
      - `large` is 1600x800
  - `benchmarkDefaultParameters` specifies the default test options
    - open `developer.html` instead to customize the options
  - `Suites` contains a single `Suite` (unless in `developer.html`)
    - the `MotionMark` suite consists of several tests
      - `Multiply` uses `core/multiply.html`
      - `Canvas Arcs` uses `core/canvas-stage.html?pathType=arcs`
      - `Leaves` uses `core/leaves.html`
      - `Paths` uses `core/canvas-stage.html?pathType=linePath`
      - `Canvas Lines` uses `core/canvas-stage.html?pathType=line&lineCap=square`
      - `Images` uses `core/image-data.html`
      - `Design` uses `core/design.html`
      - `Suits` uses `core/suits.html`
  - `_startBenchmark`
    - `ensureRunnerClient` creates `BenchmarkRunnerClient`
    - `BenchmarkRunner::runMultipleIterations` runs the tests
      - `BenchmarkRunnerClient::willStartFirstIteration` creates
        `ResultsDashboard`
      - `runAllSteps` calls `step`
        - first step creates a `BenchmarkRunnerState`
          - this is an iterator pointint to the first test of first suite
        - first test calls `_appendFrame`
          - this inserts an `<iframe>` into `<section id="test-container">`
        - `prepareCurrentTest` points the `<iframe>` to the test url and calls
          `_runBenchmarkAndRecordResults`
          - it creates `contentWindow.benchmarkClass` test class and runs it
    - `SectionsManager::showSection`
- `Benchmark::run`
- `MultiplyBenchmark` is a `Benchmark` with `MultiplyStage`
