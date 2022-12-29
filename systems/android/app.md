Android App
===========

## New process

- `ActivityManagerService` calls `Process.start("android.app.ActivityThread", ...)`
  to start a new process
  - the call asks zygote to fork and run `android.app.ActivityThread.main`
- There are `ApplicationThread` andd `ActivityThread`, run by the main thread.
  - `ApplicationThread` takes remote calls from the service, and queue
    messages.
  - `ActivityThread` handles queued messages to load apk, launch activity, and
    etc.
- An `Activity` defined by the apk is created.  An `ContextImpl` defined in the
  runtime and an `Application` defined also by the apk is attached to the
  activity.  A `Window` and `LocalWindowManager` is created.
- `Window` is an object for top-level look and behavior policy.  It usually
  creates a `View` for decoration.  When `Activity` calls `setContentView`,
  the content view is added to the decoration.
- When the activity is made visible, `WindowManagerImpl.addView` is called to
  add the decoration view.  It creates a `ViewRootImpl` and calls `setView`.
  `ViewRootImpl` talks to `WindowManagerService`.
  - If HW accel is enabled, `enableHardwareAcceleration` is called.  It calls
    `HardwareRenderer.createGlRenderer` to create the renderer.
- When the window focus changed,
  `mAttachInfo.mHardwareRenderer.initializeIfNeeded` is called
- When any event happens that requires a redraw,
  `ViewRootImpl.scheduleTraversals()` is called.  In `performTraversals`,
  - It calls `relayout` to get the `Surface`.
- In `ViewRootImpl.draw`,
  - when HW accel is disabled, it calls `surface.lockCanvas` to lock the
    canvas and calls `mView.draw` to draw the canvas.  After drawing,
    `surface.unlockCanvasAndPost` is called to post.
  - When HW accel is enabled, it calls `mAttachInfo.mHardwareRenderer.draw`

## `HardwareRenderer`

- In `initialize`, `initializeEgl` is called to initialize EGL.
  It then creates an EGL window surface and makes it current.  A
  `HardwareCanvas` is finally created with `OpenGLRenderer`.
  `mCanvas.setViewport` calls `glViewport` among others.
- When it draws a `View`, it asks the `View` to create a display list and
  draws the display list.
  - For that, the `View` calls `mHardwareRenderer.createDisplayList` to create
    a display list and record the drawing.

## `TextureView`

- A `TextureView` has a `SurfaceTexture`.  A `SurfaceTexture` can be rendered
  to as a native window.  When a new frame is rendered to,
  `OnFrameAvailableListener` is called.
- `TextureView` passes the `SurfaceTexture` to the producer.  When the
  producer produces a new frame, it updates the hardware layer (zero-copy).

## HW accel

- How does a `View` draw?
- How is a `View` drawn (by its parent)?
  - `layer = child.getHardwareLayer(); canvas.drawHardwareLayer(layer);`
  - `getHardwareLayer` will make the child create a hardware layer and draw
    itself on it.
    - For all views except `TextureView`, the child draws to an FBO
    - For `TextureView`, the child draws
- There are several ways to get a `HardwareCanvas`, which a `View` used to
  draw
  - `HardwareRenderer.create`:  It creates a `HardwareRenderer` and provides
    `HardwareRenderer.draw` to draw a `View`.  Its native
    `OpenGLRenderer`-based canvas is not exposed directly.
  - `HardwareRenderer.createDisplayList`: It creates a `DisplayList`.  To get
    the native `DisplayListRenderer`-based canvas, `start()` should be called.
  - `HardwareRenderer.createHardwareLayer(w, h, opaque)`: It creates a
    `HardwareLayer`.  To get the native `LayerRenderer`-based canvas,
    `start()` should be called.
  - `HardwareRenderer.createHardwareLayer(opaque)`: It creates also a
    `HardwareLayer`.  But it has no canvas.  Instead, calling
    `getSurfaceTexture()` returns its `SurfaceTexture`.  `SurfaceTexture` can
    be wrapped in a native `SurfaceTextureClient` to become a native window.
- `GLES20TextureLayer` is used by `TextureView`.  `GLES20RenderLayer` is used
  by all other views

package
- jarsigner -keystore ~/.android/debug.keystore <apk-file> androiddebugkey, password android

beagle
- sudo cp ~/google/mydroid/out/target/product/eee_701/system/app/Firefox.apk /data/app/org.unk.firefox.apk
- cp mplayer, qxxx.flv
- mkfs.vfat /dev/mmcblp0p1
- mke2fs /dev/mmcblp0p2
- mmcinit
- fatload mmc 0 0x80300000 uImage.bin
- setenv bootargs console=ttyS2,115200n8 noinitrd root=/dev/mmcblk0p2
  video=omapfb:mode:1280x720@50 init=/init rootfstype=ext2 rw rootdelay=1 nohz=off
- bootm 0x80300000

ActivityManagerService
- AMS: ActivityManagerService
  AT: ActivitiyThread (or ApplicationThread)
  IAM: IActivityManager
  IAT: IApplicationThread
  HR: HistoryRecord
- AMS centric view.
  IActivityManager: provide remote interface to AMS.
  IApplicationToken: provide remote interface to a HistoryRecord (activity)
  IApplicationThread: high level interface to a process, provided by remote (ApplicationThread)
  ProcessRecord: local object modeling a process (has a IApplicationThread)
- AMS manages activities, which are grouped into tasks.  Activities are provided
  by packages and run by processes.
- Starting an activity not currently run by any process involves creating a new
  managed process.
- For a process to be managed, it should ask for attaching.  AMS will bind it
  to provide info like process name, etc.
- start new activity (see also following 2 items):
  AMS.startActivity -> AMS.startActivityLocked -> find in current processes,
  tasks and HRs, AMS.startActivityLocked (with HR)  ->
  inform WMS, AMS.resumeTopActivityLocked -> AMS.startSpecificActivityLocked ->
  AMS.startProcessLocked -> unmanaged process
- manage a process:
  AT.attach -> AT IAM.attachApplication -> ATTACH_APPLICATION_TRANSACTION ->
  AMS.onTransact -> AMS.attachApplication -> AMS.attachApplicationLocked -> 
  AMS IAT.bindApplication -> BIND_APPLICATION_TRANSACTION -> AT.onTransact ->
  AT.bindApplication and queue msg -> AT.handleBindApplication -> callApplicationOnCreate
- schedule launch (create, start and resume an activity):
  AMS.attachApplicationLocked -> AMS.realStartActivityLocked ->
  AMS IAT.scheduleLaunchActivity and r's state set to RESUMED (coz andResume always true)

Activity lifecycles
- first time launch (onCreate, onStart, onResume):
  AMS IAT.scheduleLaunchActivity -> SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION ->
  AT.onTransact -> AT.scheduleLaunchActivity and queue msg ->
  AT.handleLaunchActivity -> AT.performLaunchActivity ->
  load classes, inst.callActivityOnCreate, activity.performStart
  Also in AT.handleLaunchActivity, handleResumeActivity is called to resume.
- onPause:
  AMS.startPausingLocked -> AMS IAT.schedulePauseActivity -> 
  SCHEDULE_PAUSE_ACTIVITY_TRANSACTION -> AT.schedulePauseActivity ->
  AT.handlePauseActivity -> AT.performPauseActivity (calls onPause) and
  AT IAM.activityPaused -> ACTIVITY_PAUSED_TRANSACTION -> AMS.onTransact ->
  AMS.activityPaused and state set to PAUSED
- onResume:
  AMS.resumeTopActivityLocked -> many things and AMS IAT.scheduleResumeActivity ->
  SCHEDULE_RESUME_ACTIVITY_TRANSACTION -> AT.onTransact -> AT.scheduleResumeActivity ->
  -> AT.handleResumeActivity -> performResumeActivity (calls onResume), add window to wm and others
- onStop:
  AMS.stopActivityLocked -> set HR state to STOPPING, tell WMS app invisible,
  and AMS IAT.scheduleStopActivity -> ... -> AT.handleStopActivity ->
  AT.performStopActivityInner (calls onStop), AT.updateVisibility, and
  AT IAM.activityStopped -> ... -> set HR state STOPPED
- onRestart: happens only on ActivityGroup or TabActivity
- onDestroy: 
  AMS.destroyActivityLocked -> AMS IAT.scheduleDestroyActivity and set
  HR state DESTROYING -> ... -> AT.handleDestroyActivity ->
  AT IAM.activityDestroyed -> ... -> AMS.activityDestroyed ->
  removeActivityFromHistoryLocked -> remove HR from history and WMS, set STATE DESTROYED.

new process
- ActivityManager calls Process.start() spawns a new process to run
  android.app.ActivityThread::main().  Process.start() -> startViaZygote() ->
  zygoteSendArgsAndGetPid() to ask zygote to spawn
- zygote calls ZygoteConnection::runOnce to spawns and the child enters
  handleChildProc
- Since Process::start() sets --runtime-init, RuntimeInit::zygoteInit is
  called.  In zygoteInitNative, onZygoteInit is called.  Later,
  ZygoteInit.MethodAndArgsCaller exception is thrown and
  android.app.ActivityThread::main runs.
- onZygoteInit called is app_process's, which calls ProcessState::self and
  ProcessState::startThreadPool.  Main "Binder Thread #0" is started
  and enters IPCThreadState::joinThreadPool.
- stack up to onCreate
  android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2268)
  android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2284)
  android.app.ActivityThread.access$1800(ActivityThread.java:112)
  android.app.ActivityThread$H.handleMessage(ActivityThread.java:1692)
  android.os.Handler.dispatchMessage(Handler.java:99)
  android.os.Looper.loop(Looper.java:123)
  android.app.ActivityThread.main(ActivityThread.java:3948)
  java.lang.reflect.Method.invokeNative(Native Method)
  java.lang.reflect.Method.invoke(Method.java:521)
  com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:782)
  com.android.internal.os.ZygoteInit.main(ZygoteInit.java:540)
  dalvik.system.NativeStart.main(Native Method)

Threads
- the main thread is loopping, reading from MessageQueue
- messages are dispatchMessage to target Handler, and handleMessage is called
- ???
- binder message goes into message queue?
- ActivityManager calls IApplicationThread's scheduleDestroyActivity on a IBinder
  and sends SCHEDULE_FINISH_ACTIVITY_TRANSACTION
- ApplicationThreadNative receives SCHEDULE_FINISH_ACTIVITY_TRANSACTION, rends out
  the IBinder and calls ApplicationThread's scheduleDestroyActivity, and sends
  H.DESTROY_ACTIVITY to the message queue.

android.app.ActivityThread
- prepareMainLooper: a thread-local new Looper and a new Queue are created
- ActivityThread is created (an object, not a thread) and attach()ed
- main thread is into loop() to parse Queue.
- after attaching, an IApplicationThread is ATTACH_APPLICATION_TRANSACTION to activity manager

android.app.Activity
- extends ContextThemeWrapper
- has an android.view.Window
- a context is attached in attach(), which is a new ApplicationContext()
  created by ActivityThread.

context
- in Activity.attach, a Window is ctor with activity itself
- a Window's decor is ctor with Window's context.
- in Activity.handleResumeActivity, the window's decor is addView to WM
- in WM.addView, a ViewRoot is ctor with the view's context
- In Activity.onCreate, other views are usually ctor with activity itself
- when getSystemService,
  ContextThemeWrapper: return local "layout_inflater" service
  Activity: return local "window" service
  ApplicationContext: return WindowManagerImpl for "window" service
                      return xxx for xxx
- when ask a context for "window" service, mostly got activity's window's
  window manager, which is a LocalWindowManager wrapping WindowManagerImpl,
  which is a WindowManagerImpl with parameters checking.
