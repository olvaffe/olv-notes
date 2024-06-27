Window Manager
==============

## AMS and WMS

- see also android-app
- ActivityManagerService, WindowManagerService, and Activity
- When AMS receives an Intent, it launches an Activity, represented
  by HistoryRecord, and assigns it a token and a TaskRecord.
  AMS tells WMS about the new token, and tells Activity to attach
  the token, which causes the Window and ViewRoot of the Activity to
  be created.  Activity then talks to WMS with the token.
- AMS maintains mHistory, an ordered ArrayList of HistoryRecord, which represents Activities
  HistoryRecords (activities) are grouped by tasks.  Each HR is assgined a TaskRecord.
  Two HRs with the same TaskRecord are said to be in the same task.
  HistoryRecord implements IApplicationToken.
  HistoryRecord is passed to WMS as AppWinToken
- AMS manages activity lifecycle, WMS manages activity's surfaces, and ViewRoot
  manages activity's View.
  e.g. deep in settings, press home, launch settings again and still deep.  how?
  ans: AMS calls WMS's moveAppTokensToTop
- AMS keeps WMS's mAppTokens in sync with its mHistory, a list of activities.
  WMS keeps its mWindows corresponds to its mAppTokens.

## AMS -> WMS

- startActivityLocked (many args):
  three things to resolve: create new process? create new task? create new activity?
  in case of bringing an existing task to front, nothing is created
- startActivityLocked (a HR):
  If starting in an existing invisible task, nothing to do but addAppToken() to WMS.
  If it is the first activity, nothing to do (no animation) but addAppToken()
  If the process is not yet created, showStartingIcon = true
  Calls WMS.prepareAppTransition, WMS.addAppToken, and WMS.setAppStartingWindow
  Finally, resumeTopActivityLocked is called without prev activity
- resumeTopActivityLocked:
  It is called everywhere, do not assume only for starting new activity
  But we assume starting new activity here
  If no activity at all, startActivityLocked with Launcher
  If top activity (just added one) is the resumed one, nothing but WMS.executeAppTransition
  If there is a resumed one, startPausingLocked and return! `<--` this is most common
  After the resumed one is paused, completePauseLocked calls resumeTopActivityLocked
  Now resumeTopActivityLocked is called with just paused activity as prev
  If there is prev, WMS.prepareAppTransition for `TRANSIT_TASK_OPEN`
  If next has not been launched (no app or no app.thread), startSpecificActivityLocked
  Otherwise, WMS.setAppVisibility(true) and WMS.updateOrientationFromAppTokens() to
  re-evaluate the orientation.  Then updateConfigurationLocked, scheduleSendResult,
  scheduleResumeActivity and completeResumeLocked.
- startSpecificActivityLocked:
  It calls either realStartActivityLocked or startProcessLocked.
  realStartActivityLocked scheduleLaunchActivity the activity.
  startProcessLocked starts a new process, which will finally make a remote
  call to AMS.attachApplication, which calls realStartActivityLocked.
- completeResumeLocked:
  setFocusedActivityLocked: calls WMS.setFocusedApp
  HR.resumeKeyDispatchingLocked: calls WMS.resumeKeyDispatching on itself
  ensureActivitiesVisibleLocked: 
  WMS.executeAppTransition
- start new activity: a look at WMS for what startActivityLocked did
  prepareAppTransition:
  addAppToken: add a new AppWinToken
  setAppStartingWindow: mStartingIconInTransition set to true and `ADD_STARTING` queued
  `ADD_STARTING`: mPolicy.addStartingWindow is called to add a new view "Starting com.android.settings" with the app token
                before addStartingWindow returns, the view is addWindow()ed
                then the view is set as token's startingView
  in another thread, relayoutWindow on starting view is called to
    createSurfaceLocked
    updateFocusedWindowLocked -> computeFocusedWindowLocked -> focus is launcher
    performLayoutAndPlaceSurfacesLockedInner -> resize startingView
    updateReportedVisibilityLocked
    updateOrientationFromAppTokensLocked
    sendNewConfiguration -> causes activity manager to ???
- pause current activity: when resumeTopActivityLocked is called first time:
  Nothing but startPausingLocked to pause launcher
  mResumedActivity set to null and mPausingActivity set to launcher
  After launcher paused, activityPaused is called
  In completePauseLocked, launcher is added to mStoppingActivities
  and mPausingActivity set to null
  resumeTopActivityLocked is called with launcher as prev
- start new process: when resumeTopActivityLocked is called second time:
  Since there is prev, prepareAppTransition is called
  next.hasBeenLaunched is set to true and startSpecificActivityLocked is called
  A new process is startProcessLocked()ed
- real start activity:
  when the new process calls attachApplicationLocked, ProcessRecord is attached to the thread
  realStartActivityLocked is called
  calls HR.startFreezingScreenLocked -> WMS.startAppFreezingScreen, because lots of updates are coming
  calls WMS.setAppVisibility(true)
  calls WMS.updateOrientationFromAppTokens and AMS.updateConfigurationLocked to ensure configuration and visibility
  HR.app = app
  updateLRUListLocked to adjust oom killer
  calls scheduleLaunchActivity
  calls completeResumeLocked, which WMS.executeAppTransition()

## WMS classes

- public class WindowManagerService extends IWindowManager.Stub implements Watchdog.Monitor {
    static class WMThread extends Thread {
    static class PolicyThread extends Thread {
    final class KeyWaiter {
        public class DispatchState {
    private class KeyQ extends KeyInputQueue implements KeyInputQueue.FilterCallback {
    private final class InputDispatcherThread extends Thread {
    private final class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {
    private final class WindowState implements WindowManagerPolicy.WindowState {
        private class DeathRecipient implements IBinder.DeathRecipient {
    class WindowToken {
    class AppWindowToken extends WindowToken {
    static final class DummyAnimation extends Animation {
    static final class StartingData {
    private final class H extends Handler {

## orientation

- updateOrientationFromAppTokensLocked calls a family of functions
  getOrientationFromWindowsLocked get orientation from non-app windows
  getOrientationFromAppTokensLocked get orientation from app
  computeForcedAppOrientationLocked calls above two

## transition

- prepare stage and execute stage
- in prepare stage, app being setAppVisibility(true) is put in mOpeningApps
                    app being setAppVisibility(false) is put in mClosingApps

## WindowManagerService.Session

- A Session tracks an IInputMethodClient which is provided by a client.
  It helps establish connection to SurfaceSession and InputManager.  It records
  the number of windows of a client and records the pending pointer and trackball
  event.
- A new window is created by add()ing an IWindow.  It must associate itself with
  an app token (happens early at activity attach() time) and it calls wm service's
  addWindow which associates the IWindow with a WindowState.  The window state
  is attach()ed and addWindowToListInOrderLocked().
- A window is removed by remove()ing an IWindow, which calls removeWindow.
- Many ops either calls through openTransaction, do something, and
  closeTransaction, or relayoutWindow, or performLayoutAndPlaceSurfacesLocked.

## WindowManagerService.WindowState
- mChildWindows stores a window's sub-windows
- a WS always has a mToken, but might not have mAppToken
- mBaseLayer, mSubLayer, mLayer, mAnimLayer, mLastLayer
- mAnimLayer is adjusted from mLayer by AppWinToken.

## WindowManagerService.AppWindowToken

- an activity
- a WindowToken contains a IBinder token, (maybe) AppWindowToken, and a list of WindowState
  a AppWindowToken is a WindowToken, and contains IApplicationToken, and a list of WindowState
- every window has a WindowToken
- allAppWindows and windows?

## focus

- mFocusedApp is almost not used
- computeFocusedWindowLocked returns almost the top of Window
- i.e. the top Window is almost always the focus

## layout

- performLayoutLockedInner positions all windows, setting their mFrame and insets
- openTransaction
- animation
  (applyAnimationLocked was called on app tokens and window states)
  stepAnimationLocked on every app token
  stepAnimationLocked on every window state
  probabily done by policy thus not in WMS
- Update surface
  for each window, if frame changed, setPosition and setSize and added to mResizingWindows
                   if layer, etc. changed, setAlpha, setLayer, setMatrix
                   if effect, add dim and/or blur under the window
- closeTransaction
- inform client window resized
- 

## WindowManagerService.KeyWaiter
- A key waiter waits on a single input event
- before an input event can be dispatched,
    1. target window must be found
    2. previous input event must be finished, if any
    3. if previous event is special (HOME key, e.g.), it is guaranteed to timeout
    4. dispatch process guaranteed to timeout only if failIfTimeout is true
- findTargetWindow: Find the target window for the given event.
  For pointer event, also remember windows that has `FLAG_WATCH_OUTSIDE_TOUCH` set.
  and set mMotionTarget to the target window.
- checking the return value of waitForNextEventTarget shows that previous event
  is finished if mFinished == true AND mLastWin == null
- mMotionTarget is useless, destined to be set to null if it is not null
- bindTargetWindowLocked means the event is in process.  It should be finished
  by client.
- `ACTION_MOVE` does not pass event to client directly.  Why?
  - to free from event storm?  doesn''t seem so

## WindowManagerService

- mSessions (HashSet)
- mTokenMap (HashMap), mTokenList (ArrayList), mExistingTokens (ArrayList)
- mAppTokens (ArrayList), mExitingAppTokens (ArrayList), mFinishedStarting (ArrayList),
  mOpeningApps, mClosingApps
- mWindowMap (HashMap), mWindows (z-ordered ArrayList), mResizingWindows,
  mPendingRemove, mDestroySurface, mLosingFocus, mForceRemoves
- focus: mCurrentFocus, mLastFocus, mFocusedApp

## input

- EventHub interfaces /dev/input/eventX and manages keyboard layouts (.kl).
  events are passed to KeyInputQueue, almost raw.

    If a device has any key, it is |= CLASS_KEYBOARD
                      Q key, it is |= CLASS_ALPHAKEY
    If has BTN_MOUSE, REL_X, REL_Y, it is |= CLASS_TRACKBALL
           BTN_TOUCH, ABS_X, ABS_Y, it is |= CLASS_TOUCHSCREEN
    The returned event has (devid, type, scancode, keycode, flags, value, when)
    ev.code == scancode, keycode is mapped through layout, flags is given by layout
- KeyInputQueue JNI has a EventHub.  KeyInputQueue has a thread reading events
  from the EventHub and saves them in the queue.  Events are retrieved by getEvent.
  Event hub devices are abstracted by InputDevice.
  From event hub, RawInputEvent is read.  They are aggregrated into KeyEvent
  (for keyboard) or MotionEvent (for mouse move, click, touchscreen with ACTION)
  after preprocess.
- relative MotionEvent are addBatch()ed.  It is recycled after used (finishedKey).
- window manager has KeyQ inheriting KeyInputQueue.  It is managed by
  InputDispatcher thread which gets events from the queue and dispatches to focus.
- wm dispatchTrackball with null -> ViewRoot.W.dispatchTrackball ->
    ViewRoot.deliverTrackballEvent -> IWindowSession.getPendingTrackballMove ->
- upper layer handles also key character maps (.kcm)

## touchscreen v.s. mouse

- touchsreen assume `input_event` seq:

    (ABS_X, ABS_Y, BTN_TOUCH=1, ABS_PRESSURE=1, SYN_REPORT=0) on down
    (BTN_TOUCH=0, ABS_PRESSURE=0, SYN_REPORT=0) on up
- mouse assume `input_event` seq:

    (BTN_LEFT, BTN_MIDDLE, BTN_RIGHT, REL_X, REL_Y) on all events

## what would happen if HOME pressed

- see policy
- calls launchHomeFromHotKey, which  mContext.startActivity(mHomeIntent); and sendCloseSystemWindows();
- ... performResume -> getWindow().makeActive()
- decor.setVisibility(View.VISIBLE);


## what would happen if a window is resized

- 

## what would happen if another activity is started

- 

## surface transaction

- every SurfaceSession is a SurfaceComposerClient
- java api forces transaction!  not C++!
- Surface.openTransaction -> SurfaceComposerClient::openGlobalTransacton calls
  all clients' openTransaction, which is LOCAL!
  Only one surface is supposed to be in transaction!
- Surface.closeTransaction -> SurfaceComposerClient::closeGlobalTransacton ->
  sm->openGlobalTransaction, all clients' closeTransaction, sm->closeGlobalTransaction
  In per-client closeTransaction, setState is called.

## Application Token

- An activity is an app token.  All windows must be associated with a token.
  - There might be non-app tokens.  For example, input method might add a token.
  - When a app is started, the policy might add a starting window
- An application token is visible if all child windows are displayed and are
  not animating.
  - see `updateReportedVisibilityLocked`
- A window might has child windows.
  - e.g., a `GLSurfaceView` is a child window
- `performShowLocked` shows the window and its children
  - enter animation is applied
- When a window is first drawn, `finishDrawingLocked` is called from the
  remote.  It is not displayed until `commitFinishDrawingLocked` is called.
- Before AMS does anything to apps, it calls `prepareAppTransition`.  When AMS
  thinks everything is ready, it calls `executeAppTransition`.
- `setAppVisibility` sets the visibility of an app
  - if no animation, done
  - else, it is appended to `mOpeningApps` or `mClosingApps`
- When an application token is removed, it is appended to exiting app tokens
  if there is an animation
