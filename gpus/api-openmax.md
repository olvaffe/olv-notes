OpenMax
=======

# OpenMax Overview

* OpenMax AL
  * no existing implementation
* OpenMax IL
  * may be used by OpenMax AL
  * or may be used by any native multimedia framework such as gstreamer
  * consists of components
    * source, splitter, codec, filter,, mixer, sink, and more
    * each component may be implemented by sw or hw
  * also an API to load/control/connect/unload the components
  * Bellagio is an opensource implementation
* OpenMax DL
  * consists of building blocks
  * each building block provides a basic function in domains such as signal
    processing, image processing, audio coding, image coding, or video coding
  * there are 5 application domains
    * AC - audio codecs
    * IC - image codecs
    * IP - image processing
    * SP - signal processing
    * VC - video codecs
  * ARM provides an open source implementation for their CPU cores

# IL Overview

* The user of IL is called an IL client
* IL clients interact with a centralized IL entity called the core

* IL clients use the core to load/unload the components, and to communiate with
  the components
  * all communications go through the core
* Ports
  * a component must have at least one port to claim IL compliant
  * there are four types of ports
    * video
    * audio
    * image
    * other
  * a port is either an input or an output
    * a port without input is a source
    * a port without output is a sink
* Profiles
  * base: support non-tunneled communication or proprietary communication
  * interop: base + tunneled communication
  * a component is either one of the two profiles
* States
  * UNLOADED
  * LOADED
  * WAIT FOR RESOURCES
  * IDLE: the componenet has all resources it needs
  * EXECUTING
  * PAUSED
  * INVALID: the component must be reloaded to function again
* Resources
  * static: those must be ready when a component turns IDLE
  * dynamic: those allocated later

# Tunneling

* A tunnel is between two ports of two components
  * one port is supplier port, it calls `UseBuffer` on the other port
  * the other port is non-supplier port
  * an allocator port is a supplier port that allocates buffers
  * a sharing port is a supplier port that re-uses buffers from another port on
    the same compoment

# GstOpenMax

* a gstreamer plugin that wraps many OMX IL components inside gstreamer
  elements.
* For example, gst element `omx_mpeg4dec`
  * it has class `GstOmxMpeg4Dec`, which inherits `GstOmxBaseVideoDec`, which
    inherits `GstOmxBaseFilter`, which inherits `GstElement`
  * it wraps `OMX.st.video_decoder.mpeg4` OMX IL component
  * the component is provided by `libomxil-bellagio.so.0`
  * When the gst element becomes ready,
    * it dlopens the omxil library (a singleton)
      * the plugin supports multiple omxil libraries
    * it looks for `OMX_Init`, `OMX_Deinit`, `OMX_GetHandle`, and
      `OMX_FreeHandle`.
    * it calls the init function to initialize the OMX IL
    * it calls `get_handle` function to get a handle to
      `OMX.st.video_decoder.mpeg4` and supplies callbacks
    * there are three callbacks
      * `EventHandler`: an event happened
      * `EmptyBufferDone`: input has been emptied and the app should produce more data to the input
      * `FillBufferDone`: output has been filled and the app should consume more data in the output buffer
    * the IL component is controlled through commands
      * `OMX_SendCommand(handle, OMX_CommandStateSet, OMX_StateExecuting, NULL)`
        to start the component
      * the response will be handled by the event handler
    * A IL component has many ports for different functions
      * at preparation time, the app should allocate the buffers required by the
        component
      * app should either call `OMX_AllocateBuffer` to ask the port to
        allote its buffer;
      * or, the app should allocate a buffer and call `OMX_UseBuffer` to ask the
        port to use the buffer.
      * when a port is disabled, `OMX_FreeBuffer` is called to free the buffers
    * When an application has produced enough data for an input, it calls
      `OMX_EmptyThisBuffer` to ask the port to empty the buffer
    * When an application has consumed all data of an output, it calls
      `OMX_FillThisBuffer` to ask the port to fill the buffer

# EGL

* `OMX_UseEGLImage` uses an EGLImage as a component port buffer
  * the data will not be accessible from the CPU
