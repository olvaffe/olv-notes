Android System Overview
=======================

## processes

* Processes and their sources

    /system/bin/servicemanager <- frameworks/base/cmds/servicemanager/
    /system/bin/installd <- frameworks/base/cmds/installd/
    /system/bin/mediaserver <- frameworks/base/media/mediaserver/
    /system/bin/vold <- system/core/vold/, replacing mountd
    /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server <- frameworks/base/cmds/app_process/
* After `start zygote`,

    zygote
    system_server
    android.process.acore <- packages/apps/Launcher

## init

* Order of execution

    early-init
    init
    early-boot
    boot
    queue_all_property_triggers
* `init` setups up property area by `init_property_area`
  Listens on `/dev/socket/property_service`
* when init receives `PROP_MSG_SETPROP` msg, it calls `handle_property_set_fd`
  * if ro.XXX, read-only
  * if ctl.XXX, start or stop service
  * else, update system propery area
  * finally, notify `property_changed`
* `libcutils` provides `property_get` and `property_set`
  set sends a `PROP_MSG_SETPROP` msg to property service
  get reads directly from `__system_property_get`

## zygote

* zygote is a daemon that initializes Android runtime, forks the system server,
  and then waiting for connection
* When an app is launched, the system server asks zygote to spawn a new process
  and run the app.  Because the runtime has been initialized, the startup time
  is minimized.
* zygote is started by
  `/system/bin/app_process -Xzygote /system/bin --zygote --start-system-server`
* It will call
  `AndroidRuntime::start("com.android.internal.os.ZygoteInit", true)` defined by
  `libandroid_runtime.so`
  * In `startVm`, `JNI_CreateJavaVM` is called to create the VM (the same process)
  * In `startReg`, all native methods defined by the rumtime are reigserted to
    the VM
  * VM's `CallStaticVoidMethod` callback is called.  It is defined by
    `dalvik/vm/Jni.c:CALL_VIRTUAL` macro.  The callback will call to
    `dvmCallMethodV` and transition from C to Java.
* `com.android.internal.os.ZygoteInit.main`
  * `AndroidRuntime::start` runs the `main` method of the given class
  * it first creates a socket for connections (for process spawn)
  * it then preloads the runtime (common java classes for apps)
  * after forking the system server, it goes to sleep and listens for
    connections
* Spawning
  * zygote spawns app processes by calling `ZygoteConnection::runOnce`
    * it calls `forkAndSpecialize` to fork
    * parent calls `handleParentProc` quickly and sleeps again
    * child calls `handleChildProc` and never returns
      * `--runtime-init` is usually given, so `handleChildProc` calls
        `RuntimeInit::zygoteInit`
  * zygote spawns system server by calling `startSystemServer`
    * it calls `forkSystemServer` to fork
    * the parent returns
    * the child calls `handleSystemServerProcess` and never returns
      * `RuntimeInit::zygoteInit` is also called here
  * `RuntimeInit::zygoteInit` invokes `main` method of the given class
    * for apps, the class is `android.app.ActivityThread`
    * for system server, the class is `com.android.server.SystemServer`
* `ActivityManagerService` connects to zygote when there is an activity launched
  * it does so by calling `Process::start` with `android.app.ActivityThread`
    * the activity thread will talk back to AMS for the app's activity class
      name after spawned
  * for details, see `Process::startViaZygote`

* Below are old notes
* Class `Process`, `ZygoteConnection`, `ZygoteInit` and `RuntimeInit`
  * Call `Process::start` to start a new process (or thread if no zygote)
  * It sends `--runtime-init --setuid=xxx --setgid=xxx` to zygote.
  * `ZygoteConnection::runOnce` reads the options, does some checks, and forks the child.
  * Depending on whether `--runtime-init` is given, `RuntimeInit::zygoteInit` or
    `ZygoteInit::invokeStaticMain` is called.  Both methods requires a class
    name to start and throws `ZygoteInit.MethodAndArgsCaller` instead of
    returning to clear the stack frames.
  * `RuntimeInit::zygoteInit` runs the specified java class's main function
  * It calls `zygoteInitNative` to allow native functions like `onZygoteInit` to
    be executed.
* `ZygoteInit::main`
  * The Zygote.
  * It `startSystemServer` and `runSelectLoopMode`.
  * When a spawn request comes, it calls `ZygoteConnection::runOnce` and catch
    `ZygoteInit.MethodAndArgsCaller`.
  * The spawned system server starts from class
    `com.android.server.SystemServer` through `RuntimeInit::zygoteInit`.
  * `startSystemServer` is similar to `ZygoteConnection::runOnce`.  One
    important difference is that it skips the checks `runOnce` do.  They include
    security checks like restricted capabilities.
  * the default capabilities of system server is 121715744, which corresponds to

      CAP_KILL
      CAP_NET_BIND_SERVICE
      CAP_NET_BROADCAST
      CAP_NET_ADMIN
      CAP_NET_RAW
      CAP_SYS_MODULE
      CAP_SYS_BOOT
      CAP_SYS_RESOURCE
      CAP_SYS_TIME
      CAP_SYS_TTY_CONFIG
* app_process: AndroidRuntime, AndroidRuntime::start, JNI_CreateJavaVM,
  AndroidRuntime::startReg, (c -> jni -> java) com.android.internal.os.ZygoteInit::main
* In ZygoteInit main function,
  preloadClasses, preloadResources, startSystemServer, runSelectLoopMode
* startSystemServer spawns the system_server and logs "System server process blah has been created"
  It calls RuntimeInit.zygoteInit with args --runtime-init --nice-name=system_server \
  com.android.server.SystemServer
* zygote prints "Accepting command socket connections" and enters runSelectLoopMode.
  For each connection, a ZygoteConnection is created and its runOnce is called.
* zygote does not use binder! (see /proc/<pid>/fd)
  Thus, it does not call ProcessState!
* In activity manager's (part of system server) processNextBroadcast, it checks
  the existence of target process.  If no one exists, it calls
  startProcessLocked which finally calls Process.start("android.app.ActivityThread")
  to spawn a new process.

## system_server

* zygote calls SystemServer::main -> load libandroid_servers.so, init1 -> system_init
* system_init
  SurfaceFlinger::instantiate, 
  if simulator:
    AudioFlinger::instantiate
    MediaPlayerService::instantiate
    CameraService::instantiate
  "starting Android runtime"
  AndroidRuntime::getRuntime
  "starting Android services"
  runtime->callStatic("com/android/server/SystemServer", "init2"): 
* SystemServer::init2 -> ServerThread::run, and a bunch of services started
* when in zygoteInit, (app_process's) onZygoteInit is called.  This is the
  first time ProcessState is used and it opens /dev/binder.

## servicemanager

* the binder context manager.  With ((void*)0) as its id.
* provide services, which are (string16, object) pairs
* transaction is IPC over binder
* binder_txn is payload of BR_TRANSACTION and BR_REPLY: target, cookie, code, flags, blah
* binder_io: helper for parsing the data
* binder_send_reply BC_FREE_BUFFERs the tranaction data first and BC_REPLY a reply
* for each newly added service, servicemanager
  store the (string16, object) in a list
  BC_ACQUIRE object's pointer
  BC_REQUEST_DEATH_NOTIFICATION the object's pointer

* linux kernel
  * no glibc, no standard utilities
  * alarm
  * ashmem: android shell memory driver
  * binder: open binder based ipc driver
  * power management
  * low memory killer
* native libraries
  * bionic c: small, no locales
  * webkit, opencore, sqlite
  * surface flinger: system-wide composer
    * sufaces passed as buffers via binder
    * can use opengl es and 2d hw accel. for its compositions
    * double buffering using page-flip
  * audio flinger
  * HAL (which are dlopen("/system/lib/libxxx.so") at runtime)
* dalvik
  * core libraries
* app framework (provide java lang, in jni)
  * activity manager: launcher
  * package manager: apk
  * window manager: z-order of windows
  * resource manager: image, audio
  * content providers: contacts, etc.
  * view system: widgets like button, table, frame, etc.
  * hw services: LocationService, TelephonyService, BluetoothService, WiFi Service, USB Service, Sensor Service

physiology
* kernel -> init -> usbd, adbd, debuggerd, rild (radio interface layer)
* then zygote
  * every app runs in its own process, and we do not want cold start of VM
  * initialize dalvik vm instance
  * load classes and listen on socket for req. to spawn VMs
  * forks on request to create VM
  * copy-on-write
* then runtime -> service manager (route req. to proper service)
* when service manager is initialized, it asks zygote to spawn system server
* system server -> surface flinger and audio flinger -> activity manager, package manager, etc.
* to sum up, there are
  * init process
  * some daemon processes
  * runtime process
  * zygote
  * system server
* finally, zygote is asked to spawn HOME process

layer interaction
* app -> runtime service -(JNI)-> native service binding -(dlopen)-> hal -> kernel
  * e.g., app -> location manager service, gps location provider -(JNI)-> gps location provider -(dlopen)-> libgps.so -> kernel


APK
* a APK provides components, like activities or content provider, which are run in one process
* a task (an application) is a collections of activities, which may span processes.
* an activity is a concrete class
* starting up: onCreate, onStart/onRestart, onResume (has focus)
* normal exec: onFreeze (likely to be shutdown, e.g., user is filling in a form, onFreeze should save what the user has filled, but should not commit to db), onPause (no longer has focus, commit)
* shutting down: onStop/onDestroy, usually not to be called
* each apk is given an unique ID (user id is per apk, not per user)
* only init, zygote, and runtime run as root

BINDER
* binder binds kernel and a process
* PROCESS: binder -> parcel -> parcelable (an interface a class could impl.) -> bundle, custom objects
* a parcelable is a class which can  marshal its to state to something binder could handle -- namely, a parcel
* bundle are typesafe containers of primitives

