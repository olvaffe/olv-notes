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

