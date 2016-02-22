Compiz
======

## Analysis of `drawWindow`

* Every window is a redirected X window
  * `XCompositeRedirectWindow` is called such that the app draws to an
    offscreen pixmap.
  * `XCompositeNameWindowPixmap` is called to get a handle to the offscreen
    pixmap.
  * When the contents of the offscreen pixmap are changed, compiz receives
    a damage event.  The event is reported to the screen to so that the damaged
    regions will be updated.
    * If screen should return true in its `damageWindowRect` if it handles the
      damage event.  Otherwise, a default handler is called.
  * compiz tries to keep windows always redirected, even on unmap or resize.
* Every window has an associated texture object
  * `GLXCreatePixmap` is called to create a `GLXPixmap` over the offscreen
    pixmap.
  * A texture object is created and the glx pixmap is `GLXBindTexImage` to it.
  * When `strictBinding`, the glx pixmap is bound only when it is being drawn.
    When no strict binding, it is bound most of the time.
  * When a window is resized or unmapped, `releaseWindow` is called. It destroys
    the associated texutre object, which is recreated on demand when needed.
* Every window has a geometry
  * Before drawing, `addWindowGeometry` is called to determine the vertices of
    the window, and how to draw the window.
  * It writes the new vertices (position, tex coords, etc.) to the window's
    vertex buffer, `w->vertices`, and maybe writes indices to `w->indices`.
  * It installs a `drawWindowGeometry` that will later draw the window according
    to the geometry.
* Finally, `drawWindowTexture` is called to draw the window with its texture
  object.
  * It `enableTexture` to make the window's texture object active.
  * It calls `drawWindowGeometry` installed in adding the geometry.
  * And it disable the texutre immediately.
  * The steps are performed with a fragment program, or with up to 4 texture
    units all binding to the window's texture object.

## Painting

* In the event loop, if a screen is damaged, its `paintScreen` is called.
  Before and after the painting, its `preparePaintScreen` and `donePaintScree`
  are called.
* A screen might have multiple outputs (randr?).  `paintOutput` is called on
  each output when `paintScreen`.  It does some transformations and calls
  `paintOutputRegion`.
* In `paintOutputRegion`,
  * `paintBackground` is called to paint the background.
  * `paintWindow` is called to paint each window.
  * Finally, `paintCursor` is called to paint cursors. (MPX?)

## Scale plugin

* In `scaleInitDisplay`,
  * wrapper `scaleHandleEvent` is created
* In `scaleInitScreen`,
  * wrapper `scalePreparePaintScreen` is created
  * wrapper `scaleDonePaintScreen` is created
  * wrapper `scalePaintOutput` is created
  * wrapper `scalePaintWindow` is created
  * wrapper `scaleDamageWindowRect` is created
* lifecycle
  * `SCALE_STATE_NONE`
  * `SCALE_STATE_OUT`: initiating
  * `SCALE_STATE_WAIT`
  * `SCALE_STATE_IN`: terminating
  * `SCALE_STATE_NONE`
* In `scaleHandleEvent`,
  * if a window is unmapped or destroyed, `scaleWindowRemove` is called
  * if the screen is grabbed, `scaleMoveFocusWindow` is used to handle up,
    down, left, and right key events
  * if the screen is grabbed, `scaleSelectWindowAt` is called to see if any
    window is hovered.
  * if a window is clicked, it is focused and `scaleTerminate` is called.
* When scale is triggered, `scaleInitiateCommon` are called
  * all windows that should be thumbnailed are remembered
  * `layoutSlotsForArea` is called to arrange slots
  * every window is assigned a slot id in `findBestSlots`
  * assign slots to windows in `fillInWindows`
  * the state is set to `SCALE_STATE_OUT`
  * and the screen is damaged
