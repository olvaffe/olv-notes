Application Fundamentals
- http://developer.android.com/guide/topics/fundamentals.html
- Every .apk is an app, has its own process, user id.
- an app has components, which falls in 4 category: Activity, Service, BroadcastReceiver, and ContentProvider
- when some component(s) of an app is needed by another app, it is launched
- ContentProvider is activated by ContentResolver.  The others are activated by Intent.
- Context.startActivity to start an activity with intent, and the activity's getIntent is called.
- Context.startService to start a service with intent, and the serivce's onStart is called;
  or Context.bindService to bind a service with intent, and the service's onBind is called.
- Context.send{,Ordered,Sticky}Broadcast to broadcast an intent, and all interested broadcast receivers' onReceive is called.
- .apk has AndroidManifest.xml, and others.  AndroidManifest.xml is used by android to, among others, learn about a package's components.
- components not listed in manifest are never run???
- manifest should also list intent-filters, otherwise, it won't receive any intent, unless it is dedicated for the app.
- a task is a stack of activities.  the bottom activity is usually started by app launcher.
- components of an app could be configured to run in different processes
- components of two or more apps could be configured to run in on process.
- component creation is done and system calls to the component are dispatched by the main thread of the process.

android.view
- Window: local abstract base class for a top-level window look and behavior
          policy.  setContentView to set a view.
	  The only implementation is PhoneWindow, which implementation the
          decorations we seen on the screen, with ViewGroup in the root.
- ViewRoot: Talk to wm service.  the top level view is local Window's decor.
- View: widget.  view start drawing after it receives onAttachedToWindow.
- ViewGroup: widget layout, which is also a View
- Surface (impl Parcelable): Handle on to a raw buffer that is being managed by the screen compositor.
- at some point, a view might not have a surface. Or, a surface might
  have a canvas without backing store. Surface are resources allocated by
  window manager service.  The are allocated when relayout.
- View treats a certain sequence of key or touch events as click, and invokes
  performClick, which by default play a sound and calls listener's onClick.
- simpilified version of draw(): draw background, call onDraw, and dispatchDraw
- ViewRoot -> decor's draw() -> ViewGroup's dispatchDraw -> drawChild visible children
- a view is attached/disattached to IWindow, which always has a Surface, but might be invalid.
  attachInfo.mWindowToken is the IWindow.asBinder().

Client <-> WM
- This is MY GUESS!!
- IWindow, IWindowManager, and IWindowSession for IPC to ViewRoot.W,
  WindowManagerService and WindowManagerService.Session.
- There is PhoneWindow extending (abstract) Window and PhoneWindowManager
  implementing WindowManagerPolicy interface.
  WindowManagerImpl implements WindowManager interface.
- both client and service has WindowManagerImpl.
- It all starts with Activity.attach calls PolicyManager.makeNewWindow which
  returns Window.  Then window calls setWindowManager with NULL, which means
  WindowManagerImpl.getDefault().
  Call seq: handleLaunchActivity -> performLaunchActivity -> create an
            activity, attach and callActivityOnCreate -> handleResumeActivity
- When the developer calls setContentView in onCreate, for PhoneWindow, a
  decorator is installed.  A subclass of FrameLayout is generated as decor and
  a content parent is generated from XML layout file, which might be decor itself.
  The developer's view is addView() to content parent.
- Later when handleResumeActivity, win.getDecorView is called and decor is wm.addView()ed.
  wm.addView creates a new ViewRoot and decor is set as root's view.
- When ViewRoot is created, it asks service manager for InputMethodManager and
  IWindowSession.  sess.openSession is called to connect to wm service.
  A ViewRoot.W is created which extends IWindow.Stub.
  In setView, requestLayout is called for the first time and ViewRoot's IWindow
  is add()ed to session.  The layout request will make performTraversals be called later.
- In performTraversals, session->relayout is called and surface is created by wm service.

android.view.ViewRoot
- ViewRoot is not a View, but it implements ViewParent.
- A ViewRoot implements the needed protocol between View and
  WindowManagerService through its mWindow (of class ViewRoot.W)
- On created, WindowManagerService::openSession is called to connect
  to WindowManagerService
- When ViewRoot's View is set, mWindow is added to WindowManagerService.
- Through the use of session, ViewRoot asks for relayout, pending keys, setInsets, etc.
- Through the use of mWindow, WMS notifies resized, dispatch{Key,Pointer,Trackball},
  visibility, focus change, and asks for debug info.
- Through ViewRoot's public functions, the its View asks for traversal, etc. 
- The main function is performTraversals.  Most events modify the state and schedule
  traversal.  Traversals means ask the View for the measured size and negotiate with
  WMS.  Ask View to layout according to the size allocated by WMS.  Ask the View to draw.
- If the first time, desired size is set to display size, attach info is
  updated and View is dispatchAttachedToWindow()
- mWinFrame is given by resized event, mWidth is returned by relayout.
  mWinFrame gives the allocated size.  Content inset (offsets to frame) gives
  the area contents are visible.
- distinguish view focus and window focus, view visibility and app visibility.
  The latters are assigned by WMS.
- ViewRoot traces transparent region, the region that won't be drawn by its view.
  This region is setTransparentRegion to WMS.

ViewRoot DEBUG_LAYOUT
- Measuring com.android.internal.policy.impl.MidWindow$DecorView@984ae0d8 in display 800x600...
  Laying out com.android.internal.policy.impl.MidWindow$DecorView@984ae0d8 to (800, 600)
  relayout: frame=[0,0][800,600] content=[0,25][0,0] visible=[0,25][0,0] surface=Surface(native-token=136659088)
- Measuring com.android.server.status.StatusBarView@985532b0 in display 800x25...
  Laying out com.android.server.status.StatusBarView@985532b0 to (800, 25)
- Measuring com.android.internal.policy.impl.MidWindow$DecorView@98575f70 in display 800x600...
  Resizing Handler{9857a8f0}: w=720 h=190 coveredInsets=[0,0][0,0] visibleInsets=[0,0][0,0] reportDraw=true
  relayout: frame=[40,217][760,407] content=[0,0][0,0] visible=[0,0][0,0] surface=Surface(native-token=136590328)
  Laying out com.android.internal.policy.impl.MidWindow$DecorView@98575f70 to (720, 190)
  Measuring com.android.internal.policy.impl.MidWindow$DecorView@98575f70 in display 800x600...
  Laying out com.android.internal.policy.impl.MidWindow$DecorView@98575f70 to (720, 190)
- menu is a window too
  Measuring com.android.internal.policy.impl.MidWindow$DecorView@9e85ab58 in display 800x600...
  Resizing Handler{9e7f8ba0}: w=800 h=77 coveredInsets=[0,0][0,0] visibleInsets=[0,0][0,0] reportDraw=true
  relayout: frame=[0,523][800,600] content=[0,0][0,0] visible=[0,0][0,0] surface=Surface(native-token=136592936)
  Laying out com.android.internal.policy.impl.MidWindow$DecorView@9e85ab58 to (800, 77)
  Measuring com.android.internal.policy.impl.MidWindow$DecorView@9e85ab58 in display 800x600...
  Laying out com.android.internal.policy.impl.MidWindow$DecorView@9e85ab58 to (800, 77)

android.view.Window
- other parts can call win.setCallback to gives it a callback
- callbacks, usually being set to activity itself
  onCreatePanelView: if this returns null...
  onCreatePanelMenu: PhoneWindow will create a default menu for you and let you know
  onPreparePanel:

android.view.SurfaceView
- a view with its own surface
- a subwindow of its ViewRoot (see onAttachedToWindow and TYPE_APPLICATION_MEDIA)
- negotiate with WMS itself
- has a anon subclass implementing SurfaceHolder
- has a subclass MyWindow implement IWindow.Stub
- on draw, clear the canvas
- updateWindow
  layout type is set to TYPE_APPLICATION_MEDIA
  layout token is set to viewroot attached window token

android.view.View
- when a view is asked to draw on a canvas,
  1. draw background
  2. save layers (optional)
  3. draw view's content (calls onDraw)
  4. draw childen (calls dispatchDraw)
  5. retore layers (optional)
  6. draw decorations (calls onDrawScrollBars)
- view is asked to draw when invalidated?

android.view.animation
- Animation is done by asking its transformation at certain time and apply the transformation
- View animation is done by view.startAnimation(a).  With an animation installed,
  its parent ViewGroup will animate the view (for simple animation?).
- When a ViewGroup is dispatchDraw and draws a child with animation:
  Inform child by onAnimationStart if first time
  Calls a.getTransformation to get transformation to apply
  Apply transformation
  Calls child.draw()
  Inform child by onAnimationEnd if no more transformation
- there are also layout animation and frame-by-frame animation
