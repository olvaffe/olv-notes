Steam
=====

## Troubleshooting

- `pacman -S steam`
  - `pacman -S ttf-liberation` if ui text is garbled
  - `pacman -S lib32-systemd` if no network

## Official `steam_latest.deb`

- `/usr/bin/steam` is a symlink to `/usr/lib/steam/bin_steam.sh`
  - it bootstraps `~/.local/share/Steam`
    - `install_bootstrap` untars
      `/usr/lib/steam/bootstraplinux_ubuntu12_32.tar.xz` to the directory
    - the tarball is also copied over (and can be used to re-bootstrap)
  - it creates `~/.steam`
    - `~/.steam/steam` is a symlink to `~/.local/share/Steam`
  - it runs `~/.local/share/Steam/steam.sh`
- `/usr/bin/steam` execs `~/.local/share/Steam/steam.sh`
  - it checks host package dependencies
    - `/usr/bin/steamdeps ~/.local/share/Steam/steamdeps.txt`
  - it creates various symlinks in `~/.steam`
  - it sets up steam runtime
    - `STEAM_RUNTIME=~/.local/share/Steam/ubuntu12_32/steam-runtime`
    - the runtime is unpacked from the bootstrap tarball
      - source at <https://github.com/ValveSoftware/steam-runtime>
    - `setup.sh` uses `zenity` to display a `Updating Steam runtime
      environment...` dialog
  - it sets up `LD_LIBRARY_PATH` to use the libraries from the runtime
    - with the help of
      `~/.local/share/Steam/ubuntu12_32/steam-runtime/run.sh --print-steam-runtime-library-paths`
  - it finally invokes `~/.local/share/Steam/ubuntu12_32/steam`
- `~/.local/share/Steam/ubuntu12_32/steam`
  - it is the proprietary steam client
  - if `DEBUGGER=gdb` is set, it is run under gdb
  - it downloads/updates all packages
    - bootstrap is a minimal environment to run the client
    - the client downloads more packages to `~/.local/share/Steam/package`

## steam client

- for command line options
  - <https://developer.valvesoftware.com/wiki/Command_Line_Options#Steam_.28Windows.29>
  - `-silent` starts steam client in the background
  - `-shutdown` kills steam client
  - `-login`
  - `-install`
  - `-console`
  - `-applaunch`
- apps are downloaded to `~/.local/share/Steam/steamapps`
- app database
  - the database is at `~/.local/share/Steam/appcache/appinfo.vdf`
  - VDF stands for Valve Data Format
  - `pip install steam` to get a parser
    - or, simply visit `https://steamdb.info/app/<appid>/config/`
  - take appid 570 for example
    - 'installdir': 'dota 2 beta'
    - 'executable': 'game/dota.sh'
    - 'arguments': '+engine_experimental_drop_frame_ticks 1 +@panorama_min_comp_layer_dimension 0 -prewarm_panorama'
- user data
  - `userdata/<userid>/config/localconfig.vdf` contains user configs
  - e.g., custom launch options

## Steam Runtimes

- <https://gitlab.steamos.cloud/steamrt>
- v1, scout
  - <https://gitlab.steamos.cloud/steamrt/steamrt/-/tree/steamrt/scout>
    - <https://gitlab.steamos.cloud/steamrt/scout>
    - <https://steamdb.info/app/1070560/>
    - <https://github.com/ValveSoftware/steam-runtime>
  - v1 is `LD_LIBRARY_PATH`-based
  - manual
    - `~/.local/share/Steam/ubuntu12_32/steam-runtime/run.sh
      ~/.local/share/Steam/steamapps/common/<installdir>/<executable>
      <arguments>`
    - `installdir`, `executable`, and `arguments` are from the app database
- v2, soldier
  - <https://gitlab.steamos.cloud/steamrt/steamrt/-/tree/steamrt/soldier>
    - <https://gitlab.steamos.cloud/steamrt/soldier>
    - <https://steamdb.info/app/1391110/>
  - v2 is container-based
    - debian 10
    - for use with proton 5.13, 6.3, and 7.0
  - manual
    - `~/.local/share/Steam/steamapps/common/SteamLinuxRuntime_soldier/run
      ~/.local/share/Steam/steamapps/common/<installdir>/<executable>
      <arguments>`
- v3, sniper
  - <https://gitlab.steamos.cloud/steamrt/steamrt/-/tree/steamrt/sniper>
    - <https://gitlab.steamos.cloud/steamrt/sniper>
    - <https://steamdb.info/app/1628350/>
  - v2 is also container-based
    - debian 11
    - for use with proton 8.0
- <https://gitlab.steamos.cloud/steamrt/steam-runtime-tools>
  - tools included in all runtime v2 and later

## Proton

- proton is installed as apps
  - since proton 5.13, it should be run under steam runtime v2
  - since proton 8.0, it shoud be run under steam runtime v3
- 7.0, <https://steamdb.info/app/1887720/>
- 8.0, <https://steamdb.info/app/2348590/>
- manual
  - `STEAM_COMPAT_CLIENT_INSTALL_PATH=~/.local/share/Steam
     STEAM_COMPAT_DATA_PATH=~/.local/share/Steam/steamapps/compatdata/<appid>
     ~/.local/share/Steam/steamapps/common/SteamLinuxRuntime_soldier/run
     ~/.local/share/Steam/steamapps/common/Proton\ -\ Experimental/proton
     waitforexitandrun
     ~/.local/share/Steam/steamapps/common/<installdir>/<executable>
      <arguments>`

## Using runtime and proton directly

- use runtime directly
  - `cp -a ~/.local/share/Steam/steamapps/common/SteamLinuxRuntime_sniper .`
  - parse runtime vdf for cmdline
    - `apt install python3-vdf`
    - `cp /usr/share/doc/python3-vdf/examples/vdf2json .`
    - `./vdf2json SteamLinuxRuntime_sniper/toolmanifest.vdf tmp`
    - we will get `/_v2-entry-point --verb=%verb% --`
  - run any cmd
    - `./SteamLinuxRuntime_sniper/_v2-entry-point --verb=waitforexitandrun -- ls`
- use proton under runtime directly
  - `cp -a ~/.local/share/Steam/steamapps/common/Proton\ 8.0 .`
  - parse proton vdf for cmdline
    - we will get `/proton %verb%`
  - prepare env
    - mkdir `prefix`
    - `export STEAM_COMPAT_DATA_PATH=$PWD/prefix`
    - `export DXVK_STATE_CACHE_PATH=$PWD/prefix`
    - `export STEAM_COMPAT_MOUNTS=`, no additional bind-mount
    - `export STEAM_COMPAT_CLIENT_INSTALL_PATH=`
    - `export PROTON_LOG=1`
  - run any cmd
    - `./SteamLinuxRuntime_sniper/_v2-entry-point --verb=waitforexitandrun -- $PWD/Proton\ 8.0/proton waitforexitandrun notepad`
- runtime internals
  - `_v2-entry-point` execs `run pressure-vessel/bin/steam-runtime-launcher-interface-0 container-runtime $@`
  - `run` execs `pressure-vessel/bin/pressure-vessel-unruntime $@`
  - `pressure-vessel-unruntime` escapes steam runtime and execs `pressure-vessel/bin/pressure-vessel-wrap $@`
  - `pressure-vessel-wrap` enters the container
    - <https://gitlab.steamos.cloud/steamrt/steam-runtime-tools/-/blob/main/pressure-vessel/wrap.1.md>
    - `env` shows that the container has these changes
      - `XDG_DATA_DIRS=/usr/lib/pressure-vessel/overrides/share:$XDG_DATA_DIRS:/run/host/user-share:/run/host/share`
      - `LD_LIBRARY_PATH=/usr/lib/pressure-vessel/overrides/lib/x86_64-linux-gnu/aliases:/usr/lib/pressure-vessel/overrides/lib/i386-linux-gnu/aliases`
      - `VK_DRIVER_FILES=/usr/lib/pressure-vessel/overrides/share/vulkan/icd.d...`
      - `LIBGL_DRIVERS_PATH=/usr/lib/pressure-vessel/overrides/lib/x86_64-linux-gnu/dri:/usr/lib/pressure-vessel/overrides/lib/i386-linux-gnu/dri`
      - `__EGL_VENDOR_LIBRARY_FILENAMES=/usr/lib/pressure-vessel/overrides/share/glvnd/egl_vendor.d/50_mesa.json`
  - `steam-runtime-launcher-interface-0`
    - <https://gitlab.steamos.cloud/steamrt/steam-runtime-tools/-/blob/main/bin/launcher-interface-0.md>
- debug
  - `PRESSURE_VESSEL_TERMINAL=tty ... bash` to get shell
    - the envvar keeps stdin open
  - `VK_LOADER_DEBUG=all` is passed through and just works
  - `VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation` is passed through and
    just works thanks to bind-mounts
  - `VK_DRIVER_FILES=...` is passed through and semi-works
    - it works inside the container
    - it does not work inside proton

## Fossilize

- build
  - `git clone --recurse-submodules https://github.com/ValveSoftware/Fossilize.git`
  - `cmake -S. -Bout -GNinja`
- capture with apis
  - `Fossilize::create_database` to create/open a `.foz` database
    - it returns a `Fossilize::DatabaseInterface` to manipulate the database
  - `StateRecorder recorder; recorder.init_recording_thread(db)` to initialize
    the recorder
  - from this point on, call `record_*` such as `record_application_info` or
    `record_sampler` to record objects to the db
  - when done, destroy the record to flush and stop the recording thread
- replay with apis
  - define app-specific `ReplayerInterface`
  - `StateReplayer replayer; replayer.parse(iface, db, ...)` to replay the
    database
- capture with `VK_LAYER_fossilize`
  - automatic create a db and invoke `StateRecorder`
  - `VK_LAYER_PATH=~/projects/Fossilize/out/layer/ VK_INSTANCE_LAYERS=VK_LAYER_fossilize`
  - db is saved to `fossilize.<hash>.<seqno>.foz`
    - `FOSSILIZE_DUMP_PATH=` to override `fossilize` prefix
- db internals
  - there are 3 database formats
    - `StreamArchive` with `.foz` suffix
    - `ZipDatabase` with `.zip` suffix
    - `DumbDirectoryDatabase` if neither
  - there are `RESOURCE_COUNT` resource types in a database
    - `RESOURCE_APPLICATION_INFO` is based on `VkApplicationInfo` and
      `VkPhysicalDeviceFeatures2`
    - `RESOURCE_SAMPLER` is based on `VkSamplerCreateInfo`
    - `RESOURCE_DESCRIPTOR_SET_LAYOUT` is based on
      `VkDescriptorSetLayoutCreateInfo`
    - `RESOURCE_PIPELINE_LAYOUT` is based on `VkPipelineLayoutCreateInfo`
    - `RESOURCE_SHADER_MODULE` is based on `VkShaderModuleCreateInfo`
    - `RESOURCE_RENDER_PASS` is based on `VkRenderPassCreateInfo2` and
      `VkRenderPassCreateInfo`
    - `RESOURCE_GRAPHICS_PIPELINE` is based on `VkGraphicsPipelineCreateInfo`
    - `RESOURCE_COMPUTE_PIPELINE` is based on `VkComputePipelineCreateInfo`
    - `RESOURCE_APPLICATION_BLOB_LINK` is ???
    - `RESOURCE_RAYTRACING_PIPELINE` is based on
      `VkRayTracingPipelineCreateInfoKHR`
  - each `StateRecorder::record_*` call records a `Vk*Info` to a json snippet,
    which is saved to the database
- `fossilize-convert-db` converts between supported database formats
  - it opens an input db for reading and an output db for writing
  - it reads all resources from the input db and writes them to the output db
- `fossilize-merge-db` merges databases
  - it opens the first db for writing and the rest dbs for reading
  - it reads all resources from the rest of the dbs and writes them to the
    first db
- `fossilize-opt` optimizes spirv
  - it opens the input db for reading and the output db for writing
  - it reads resources of certain types from the input db and replays them
    with `OptimizeReplayer`
  - `OptimizeReplayer` records all replayed resources
    - for `RESOURCE_SHADER_MODULE`, it uses `SPIRV-Tools` to optimizes spirv
      before recording
- `fossilize-list` lists all hashes of a given resource type (tag)
  - the most interesting tags should be
    - `RESOURCE_SHADER_MODULE` (4)
    - `RESOURCE_GRAPHICS_PIPELINE` (6)
    - `RESOURCE_COMPUTE_PIPELINE` (7)
    - `RESOURCE_RAYTRACING_PIPELINE` (9)
  - it opens the input db for reading
  - it reads the hashes of all resources of the given tag
  - it prints hashes (and optionally sizes) of resources
- `fossilize-rehash` recomputes hashes, used to convert an old db from old
  hash formats to new hash formats
- `fossilize-sync` synthesizes a db from raw spirv code
- `fossilize-prune` removes unused/unwanted entries from a db
- `fossilize-disasm` disassembles spirv
  - it opens the input db for reading
  - it replays all interesting resource types
  - when `--target asm`, it uses `SPIRV-Tools` to print spirv disassembly
  - when `--target glsl`, it uses `SPIRV-Cross` to print glsl
  - when `--target isa`, it uses `VK_KHR_pipeline_executable_properties` to
    dump the binary
    - for radv, this dumps `NIR Shader(s)`, `ACO IR`, and `Assembly` in text

## `fossilize-replay`

- a `fossilize-replay` process can be in one of the four modes
  - when `--progress` is specified, it runs in progress mode with
    `run_progress_process` being the main loop.  In this mode, it uses
    `ExternalReplayer` to start another `fossilize-replay` process with
    `--master-process`
  - when `--master-process` is specified, it runs in multi-process mode with
    `run_master_process` being the main loop.  `--num-threads` determines the
    number of child processes to fork.  Each child process uses
    `run_slave_process` as the main loop.
  - when `--slave-process` is specified, it runs in slave mode with
    `run_slave_process` being the main loop.  This is not used on Linux.  In
    multi-process mode, `fossilize-replay` forks and calls `run_slave_process`
    directly.
  - otherwise, it runs in normal mode with `run_normal_process` being the main
    loop.  `--num-threads` determines the number of threads of
    `ThreadedReplayer`
- when in progress mode, `ExternalReplayer` is used to start another
  `fossilize-replay` process in multi-process mode
  - `ExternalReplayer::Impl::start` sets up the shmem fd and the control fd
    that are used to control the external replayer and receive progress data
  - it specifies `--master-process`, `--quiet-slave`, `--shmem-fd`, and
    `--control-fd`
  - it forwards `--spirv-val`, `--num-threads`, `--on-disk-*`, and many other
    arguments
- when in multi-process mode,
  - `--control-fd` to specify a control fd
    - it can be used to control the master process
    - commands are handled by `handle_control_command`
    - `RUNNING_TARGET <N>` cotrols the number of child processes running
  - `--shmem-fd` to specify a shmem fd
    - it is used to hold a `SharedControlBlock` and a ring buffer
    - used for progress reporting from child processes
- when in slave mode,
  - it sets up handshakes with the master process and disables some signals
  - it then calls `run_normal_process`
- when in normal mode,
  - a `StateReplayer` is created to parse the database
  - as `StateReplayer` parses, recorded Vulkan states are replayed
    - e.g., `Fossilize::StateReplayer::Impl::parse_application_info` calls
      `ThreadedReplayer::set_application_info`
  - the main thread replays these states first
    - `RESOURCE_APPLICATION_INFO`
    - `RESOURCE_SAMPLER`
    - `RESOURCE_DESCRIPTOR_SET_LAYOUT`
    - `RESOURCE_PIPELINE_LAYOUT`
    - `RESOURCE_RENDER_PASS`
  - `start_worker_threads` is called
  - these states are collected as well
    - `RESOURCE_GRAPHICS_PIPELINE`
    - `RESOURCE_COMPUTE_PIPELINE`
  - `enqueue_deferred_pipelines` is called to convert the state replays into
    `EnqueuedWork`.
  - `enqueue_work_item` is called indirectly by `EnqueuedWork`s and the worker
    threads start replaying
  - `sync_worker_threads` waits for all worker threads to complete
- pipeline cache
  - by default, `fossilize-replay` warms up the driver's internal disk cache
    - it creates some `VkPipelineCache`s for pipeline replays
    - after replays, it destroys the caches
    - it does not save the pipeline caches anywhere
  - `--on-disk-pipeline-cache` saves the pipeline cache to a file
    - when specified, it creates a `VkPipelineCache` from the file
    - when done, it calls `vkGetPipelineCacheData` and saves the data back to
      the file
- crash report
  - in multi-process mode, when a child process crashes, `crash_handler` is
    called.
  - the handler uses a pipe to tell the master process it has crashed, and
    which pipeline was being replayed
  - `--timeout-seconds` specifies a timeout when stuck
    - by default, `sync_worker_threads` waits until all worker threads are
      done
    - when specified, and if no pipeline is replayed during the timeout value,
      `timeout_handler` is called.  This sends `SIGABRT` to the worker thread
      and triggers `crash_handler`
- misc options
  - `--replayer-cache` specifies a replay cache
    - it is a db that contains the hashes of replayed pipelines
    - this allows the replayed pipelines to be skipped on following runs
