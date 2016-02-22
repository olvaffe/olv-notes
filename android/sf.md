Android SurfaceFlinger
======================

## Protocol Overview

* `ISurfaceComposer` is the interface of `SurfaceFlinger`
  * `ISurfaceComposerClient` is the interface of a connection to
    `SurfaceFlinger`.  It can be created from `ISurfaceComposer`
    * `ISurface` is the interface of a on-screen surface.  It can be created
      from `ISurfaceComposerClient`.
      * `ISurfaceTexture` is the interface of the buffer storage of the
        surface.  It can be obtained from `ISurface`.
  * `IGraphicBufferAlloc` is the interface of a buffer allocator.  It can be
    created from `ISurfaceComposer`.  Buffers are allocated from this
    interface.
* This is how things work
  * an app creates `ISurfaceComposer` to talk to `SurfaceFlinger`.  It
    creates a connection, that is, `ISurfaceComposerClient`.  `ISurface`s can
    be created from the connection.  `ISurfaceTexture`s can be used to access
    surface buffers.
  * the app can also use `IGraphicBufferAlloc` to allocate offscreen buffers.

## Object Overview

* `SurfaceComposerClient` wraps `ISurfaceComposer` and
  `ISurfaceComposerClient`.
* `SurfaceControl` wraps `ISurface`.
* `Surface` wraps `SurfaceTextureClient` which wraps `ISurfaceTexture`
  * `Surface` is an old object, which becomes very thin after the introduction
    of `SurfaceTextureClient`
* `SurfaceTexture` wraps `ISurfaceCompoer` and `IGraphicBufferAlloc`.
  * It is used by clients to allocate offscreen buffers.  EGL can render to
    them, by wrapping `SurfaceTexture` in a `SurfaceTextureClient`.  Or sample
    from them, by using `EGLImage`.
  * It is also used by the server to allocate buffers and implement
    `ISurfaceTexture`
  * Because it implements `ISurfaceTexture`, it can be wrapped in
    `SurfaceTextureClient`

## EGL native types

* `FramebufferNativeWindow` is an `ANativeWindow`.  It is server-only.
* `SurfaceTextureClient` is an `ANativeWindow`.  It is client-only.
  * Thus, `Surface` and `SurfaceTexture` can both be viewed as native types
* `GraphicBuffer` is an `ANativeWindowBuffer`.  It is used by both clients and
  the server.

## Jave-level objects

* Java-level `ViewRootImpl` has a Java-level `Surface`, which maps to
  C++-level `Surface`.
* Java-level `SurfaceView` also has a Java-level `Surface`.
* Java-level `TextureView` has a Java-level `SurfaceTexture`, which maps to
  C++-level `SurfaceTexture.
