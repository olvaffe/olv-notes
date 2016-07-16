Light Introduction to Android Platform Graphics Internals
=========================================================

This introduction describes the key functionalities of platform graphics
components.  It is only to give an overview.  Always read the source code.

`libhardware`
-------------

The source code of `libhardware` is located at `hardware/libhardware`.  It
mainly provides

    int hw_get_module(const char *id, const struct hw_module_t **module);

to `dlopen` hardware modules.  Given `id`, `hw_get_module` searches for

 - `<id>.<ro.product.board>.so`
 - `<id>.<ro.board.platform>.so`
 - `<id>.default.so`
 - more names

in `/{system,vendor}/lib{64,}/hw`.  For example, to search for `gralloc`
module on angler, it will try

 - `gralloc.angler.so`
 - `gralloc.msm8994.so`
 - `gralloc.default.so`

Once loaded, modules cannot be unloaded.  The only method provided by
`hw_module_t` is

    int (*open)(const struct hw_module_t* module, const char* id,
                struct hw_device_t** device);

`open` opens a device.  `id` is a module-specific device name.  For example,
it should be `gpu0` for `gralloc`.

`gralloc`
---------

The `gralloc` module manage graphics buffers.  The interface is defined by
`hardware/libhardware/include/hardware/gralloc.h`.  It mainly provides

    int (*alloc)(struct alloc_device_t* dev,
                 int w, int h, int format, int usage,
                 buffer_handle_t* handle, int* stride);

    int (*free)(struct alloc_device_t* dev,
                buffer_handle_t handle);

    int (*lockAsync)(struct gralloc_module_t const* module,
                     buffer_handle_t handle, int usage,
                     int l, int t, int w, int h,
                     void** vaddr, int fenceFd);

    int (*unlockAsync)(struct gralloc_module_t const* module,
                       buffer_handle_t handle, int* fenceFd);

`alloc` allocates a graphics buffer satisfying the given properties.  It is
worth noting that `format` includes YCbCr formats and `usage` includes camera,
video encoder/decoder, hwcomposer, etc.  `free` frees the buffer.

`lockAsync` locks a region of a buffer for CPU access.  If `fenceFd` is not
negative, it will be signaled only when it is safe for the module to really
lock the buffer (e.g., when GPU is done rendering to the buffer).  Likewise,
`unlockAsync` may return a `fenceFd` that is signaled only when it is safe for
other components to access the buffer (e.g., safe for GPU to sample from).
There are also synchronous versions of `lockAsync` and `unlockAsync`.

The definition of `buffer_handle_t` is

    typedef struct native_handle
    {
        int version;        /* sizeof(native_handle_t) */
        int numFds;         /* number of file-descriptors at &data[0] */
        int numInts;        /* number of ints at &data[numFds] */
        int data[0];        /* numFds + numInts ints */
    } native_handle_t;

    typedef const native_handle_t* buffer_handle_t;

Since `int`s and `fd`s can both be serialized, this allows `buffer_handle_t`
to be sent to another process.

`hwcomposer`
------------

The `hwcomposer` module manages displays and display overlay planes.  The
interface is defined by `hardware/libhardware/include/hardware/hwcomposer.h`.

An overlay plane can have many states:

 - Z-order, which planes occulde which planes
 - geometry, position and size of a plane on the display
 - visible region, region visible on the display and occuldes other planes
 - transform, rotations and flips
 - source crop, a rectangle of pixel data that will be read and scaled to fill
   the plane
 - colorspace, how pixel values are interpreted
 - sideband stream, such as TV signal input
 - blend mode, how a plane blends with or occuldes other planes
 - plane alpha
 - solid color
 - composition type, use pixel data, sideband stream, or the solid color

Once all states of all active overlay planes of a display have been
programmed, the client is expected to validate them.  If the module cannot
support some overlay planes, it will tell the client to change the states or
to composite them manually.  After successfully validated, the client can
present the layer contents to the display.

Each overlay plane that sources pixel data has a `buffer_handle_t` and two
fences.  An acquire fence that is signaled when it is safe for the mdoule to
access the buffer.  A release fence that is signaled when it is safe for other
components to access/reuse the buffer.

There is also a retire fence associated with each presentation.  The fence is
signaled when the presentation is replaced by another one.

There can be unlimited number of overlay planes, since the module is allowed
to ask a client to composite planes manually.  This makes overlay planes
virtual.  There can also be virtual displays.  Virtual displays function as
frame producers and can be consumed by applications to record a screen, cast
to a remote display, etc.

The asynchronous nature of the module also requires a client to register three
callbacks

 - hotplug callback that is invoked when a physical display is
   connected/disconnected
 - vsync callback that is invoked when a vsync event happens on a display
 - invalidate callback that is invoked to request a new frame be presented

`libutils` and Smart Pointers
-----------------------------

Headers for `libutils` are located at `system/core/include/utils`.
`libutils`, among others, provides strong and weak pointers.

`sp` works similar to `std::shared_ptr`, except that its implementation is
more intrusive.  It requires the managed objects to provide two methods:
`incStrong` and `decStrong`.

Some classes such as `ANativeWindow` provides `incStrong` and `decStrong`
directly.  Others such as `NativeHandle` inherit from `LightRefBase` which
provides the two required methods using a `std::atomic<int32_t>`.  These are
ad-hoc, however.  Most classes simply inherit from `RefBase`.

`RefBase` provides the two required methods for `sp`.  Beyond that, it
provides required methods for `wp`.  `wp` works similar to `std::weak_ptr`,
except that its implementation is again more intrusive.  It pretty much
requires the managed objects to be derived from `RefBase`.  The trick played
by `RefBase` is to `new` a `weakref_type` in its constructor and let `wp`
manage the `weakref_type`.  This `weakref_type` can outlive `RefBase`.

`libui`
-------

The source code of `libui` can be found at
`frameworks/native/{libs,include}/ui`.  It provides some useful utility
classes:

 - `Point` for 2D integer coordinates `(x, y)`
 - `Rect` for 2D rectangles `(left, top, right, bottom)`
 - `Region` is a vector of `Rect`s

All of them are flattenable, meaning they can be serialized to a buffer, sent
to another process, and deserialized.

There are also `GraphicBufferAllocator` and `GraphicBufferMapper` that are
both wrappers to `gralloc` module.  `GraphicBufferAllocator` wraps `gralloc`'s
`alloc`/`free` while `GraphicBufferMapper` wraps `gralloc`'s
`lockAsync`/`unlockAsync`.

Finally, there is `GraphicBuffer`.  `GraphicBuffer` makes it much easier to
work with `gralloc`: allocate new graphics buffers, reallocate graphics
buffers, import graphics buffers from another process, etc.  `GraphicBuffer`
is also flattenable.

Binder and Service Manager
--------------------------

Binder is an IPC mechanism used for for in-process and cross-process RPC.
Every remote call is a transaction.  The transaction data consists of a plain
part that is copied verbatim and an array of `flat_binder_object`.  A
`flat_binder_object` can be one of these

    enum {
        BINDER_TYPE_BINDER      = ...,
        BINDER_TYPE_WEAK_BINDER = ...,
        BINDER_TYPE_HANDLE      = ...,
        BINDER_TYPE_WEAK_HANDLE = ...,
        BINDER_TYPE_FD          = ...,
    };

When a process sends its remotable object (that is, a service) in a
transaction, it refers to the remotable object as a `BINDER_TYPE_BINDER`.
When another process receives the remotable object, it instead gets a
`BINDER_TYPE_HANDLE`, which is just a `uint32_t`.  Conversely, when a process
makes a remote call to a remotable object, it refers to the remotable object
as a `BINDER_TYPE_HANDLE`.  The receiving process gets a `BINDER_TYPE_BINDER`.

When a process only has weak reference to a remotable object, it will use
`BINDER_TYPE_WEAK_HANDLE`.  The receiving process will see a
`BINDER_TYPE_WEAK_BINDER`.  It is also possible to send a file descriptor to
another process using `BINDER_TYPE_FD`.  This is how a graphics buffer is
shared with another process.

The source code of service manager is located at
`frameworks/native/cmds/servicemanager`.  Service manager is special in that
it can always be referred to by any process using `BINDER_TYPE_HANDLE` with
handle value `0`.  It provides a service that works as a name server.
Remotable objects can register themselves to service manager using well-known
names such as `SurfaceFlinger`.  Users of such remotable objects can use the
well-known names to look up the handles of remotable objects.

`libbinder`
-----------

The source code of `libbinder` can be found at
`frameworks/native/{libs,include}/binder`.  At the lowest level, it provides
`BBinder` to be inherited by remotable object classes and `BpBinder` as a
wrapper to `BINDER_TYPE_HANDLE` handle.  Both classes inherit from an abstract
class `IBinder`, which provides this method among others

    virtual status_t transact(uint32_t code,
                              const Parcel& data,
                              Parcel* reply,
                              uint32_t flags = 0) = 0;

The method is very close to the binder protocol, and it is tedious to flatten
each remote call into a `Parcel`, make a binder transaction, and unflatten the
result out of another `Parcel`.  There's got to be a better way.

A remotable object `Foo` built on top of `libbinder` is usually expected to
define these classes:

    // an abstract class that defines the interface of Foo
    class IFoo : public IInterface {
    public:
        virtual int method1(int val) = 0;
        ...;
    };

    // a class that unflattens call arguments out of a Parcel and flattens
    // call results into a Parcel
    class BnFoo : public BnInterface<IFoo> {
    public:
        virtual status_t transact(uint32_t code,
                                  const Parcel& data,
                                  Parcel* reply,
                                  uint32_t flags = 0);
    };

    // a class that implements the interface
    class Foo : public BnFoo {
    public:
        virtual int method1(int val);
        ...;
    };
     
    // a class that is the opposite BnFoo: it flattens call arguments into a
    // Parcel and unflattens call results out of a Parcel
    class BpFoo : public BpInterface<IFoo> {
    public:
        virtual int method1(int val);
        ...;
    };

The local process (that is, the service provider) implements `class Foo`.
When a remote process gets the handle to the remotable object, it wraps the
handle in a `BpBinder`, and then wraps the `BpBinder` in a `BpFoo`.  This way,
the two processes can implement/use `Foo` without knowing it is
remotable/remoted.

One caveat is `class Foo` must be thread-safe.  When a transaction arrives at
a local process, and there is no thread ready to serve it, `libbinder` would
spawn a new thread, capped at like 15 threads per process.  Methods of `Foo`
are called by those threads.

`libgui`
--------

The source code of `libgui` can be found at
`frameworks/native/{libs,include}/gui`.  `libgui` mainly defines the
interfaces between a surface composer and its clients.  The only surface
composer implemented is called `SurfaceFlinger`.

The interesting methods defined by `ISurfaceComposer` are

    virtual sp<ISurfaceComposerClient> createConnection() = 0;
    virtual void setTransactionState(const Vector<ComposerState>& state,
                                     const Vector<DisplayState>& displays, uint32_t flags) = 0;

Every application generally calls `createConnection` once to establish a
connection to the surface composer.  This gives the application an
`ISurfaceComposerClient` to

    virtual status_t createSurface(
            const String8& name, uint32_t w, uint32_t h,
            PixelFormat format, uint32_t flags,
            sp<IBinder>* handle,
            sp<IGraphicBufferProducer>* gbp) = 0;

`handle` is used to identify the surface for state changes: position, size,
z-order, crop, etc.  The application buffers all state changes into a vector
of `ComposerState` locally and calls `setTransactionState` to apply all
changes in one go.  `gbp` is used to dequeue buffers for rendering and queue
buffers back, to be consumed by the surface composer.

Applications do not make calls to the surface composer directly.
`WindowManager` that is a part of the system server makes those calls on their
behalf.  `WindowManager` does not make those calls directly either.  `libgui`
provides higher-level classes such as `SurfaceComposerClient`,
`SurfaceControl`, and `Surface` to wrap the lower-level interfaces.
`WindowManager` works with the higher-level classes.  Of them, `Surface` both
wraps `IGraphicBufferProducer` and inherits from `ANativeWindow`.  It is what
gets passed to `eglCreateWindowSurface` or `vkCreateAndroidSurfaceKHR`.

A `BufferQueue` has a producer end and a consumer end.  The two ends
implements `IGraphicBufferProducer` and `IGraphicBufferConsumer` interfaces
respectively.  When a `SurfaceFlinger` client creates a surface, a
`BufferQueue` is created by `SurfaceFlinger` and the producer end is returned
back to the client.  A client may also create a `BufferQueue` directly and
locally.  The scenario is, for example, to put decoded video frames into the
producer end and feed them to GLES pipeline on the consumer end through
`EGLImage`s.

`libGLESv*`
-----------

The source code of `libGLESv*` can be found at
`frameworks/native/opengl/libs/GLES2`.  `libGLESv*` provides dispatch
stubs for all known GL commands

    void glFoo(...) {
        gl_hooks_t::gl_t const * const _c = &getGlThreadSpecific()->gl;
        if (_c) _c->glFoo(...);
    }

`getGlThreadSpecific` returns the thread-local `const gl_hooks_t *` stored in
the TLS slot `TLS_SLOT_OPENGL_API`.  `TLS_SLOT_OPENGL_API` is like a
preallocated `pthread_key_t`, except it enables less indirections and is more
efficient.  The returned pointer is set up by `libEGL` during
`eglMakeCurrent`.

In reality, the stubs are actually written in assembly instead of the slower C
shown here.

`libEGL`
--------

The source code of `libEGL` can be found at
`frameworks/native/opengl/libs/EGL`.  `libEGL` manages vendor EGL/GLES driver
by looking for

 - `libEGL.so`
 - `libGLESv1_CM.so`, and
 - `libGLESv2.so`

in `/{vendor,system}/lib{64,}/egl`.  There was a time that there could be
multiple vendor drivers, mainly for hardware-accelerated driver and
software-based driver to coexist.  But that seems to have changed.  `libEGL`
now functions as a thin wrapper to vendor's `libEGL`, and provides some driver
independent error checks and functionalities.

The vendor driver is represented by a global variable `gEGLImpl` of type
`egl_connection_t`.  `gEGLImpl` has dispatch tables that point to vendor's
EGL/GLES entrypoints.  At `eglMakeCurrent` time, TLS slot
`TLS_SLOT_OPENGL_API` is set to the correct (v1 or v2) GLES dispatch table,
which is used by the system's `libGLESv*` to dispatch to the vendor driver.

(`libEGL` provides 256 dispatch stubs in `gExtensionForwarders` for
`eglGetProcAddress` to return for unknown extension functions.  Since `libEGL`
can support only a vendor driver now, that may not be necessary anymore.

`libEGL` also installs a dummy `gl_hooks_t` before `eglMakeCurrent`.
`getGlThreadSpecific` should never return `NULL`.  I wonder why dispatch stubs
in `libGLESv*` check for `NULL`.  With the ability to support multiple vendor
drivers gone, I am not entirely sure about the value of dispatch stubs.

There also appears to be dead code here and there.  For example,
`GL_EXT_debug_marker` could be removed since GLTrace is deprecated.)

SurfaceFlinger
--------------

The source code of `SurfaceFlinger` can be found at
`frameworks/native/services/surfaceflinger`.  It uses everything from above to
composite windows/surfaces/layers and display the final results.

Command Line Tools
------------------

The source code of `start` and `stop` is located at `system/core/toolbox`.
They speak with `init` (pid 1) over system properties and start/stop init
services respectively.

There are many useful command line tools whose source can be found at
`frameworks/{native,base,av}/cmds`.

`service` talks with the special binder service, service manager, to list,
check, or even call remote services registered with the service manager.

All `libbinder`-based services support several methods: dump, shellCommand,
etc.  dump can be invoked using `dumpsys`.  shellCommand can be invoked using
`cmd`.

`screenrecord` can record the display to a `.mp4` file.  It does so by
creating a virtual display and feeding all frames into a media codec.
