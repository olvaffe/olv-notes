OpenWF
======

## Display & Composition

* A composition device is either fully capable or off-screen only.  A compositor
  wants a fully capable device.  If there is only off-screen device, the
  compositor must use a display device to output the content of the off-screen
  buffer.
* There is no on-screen only composition device.  Such device is called a
  display device.

## Composition

* The concept is quite simple
* `wfcCreateDevice`
* `wfcCreateOnScreenContext`
  * create a context that renders to a screen
* `wfcCreateOffScreenContext`
  * create a context that renders to another stream
* `wfcCreateSourceFromStream`
* `wfcCreateMaskFromStream`
* `wfcCreateElement`
  * describe how a image source is sampled and how is it mapped to the target
    image
* `wfcInsertElement`
* `wfcRemoveElement`
* `wfcActivate` activates autonomous composition
* `wfcCompose`
  * sources and masks are called image providers
  * destination screen or stream are called target images
  * This is an async call.  All target images and image providers are still used
    until completion, which can be waited with `wfcFence`
* `wfcFence`

## Display

* The concept is more complex
  * Device is a display control hardware, or a DRM device
  * A port is a connector
    * a CRTC and an encoder is automatically matched
  * A display hardware is a LCD or CRT monitor
  * A pipeline is the stages before the data are fed into the connector
    * transparency, layering, etc.
* Comparing to on-screen composition,
  * a `WFDSource` is a `WFCSource`
  * a `WFDPipeline` is a `WFCElement`
  * the fact the pipelines are limited and elements are unlimited says
    * wpc needs to perform off-screen composition of all elements
    * then wpc passes the result to a pipeline
* `wfdCreateDevice`
* `wfdCommitDevice`
* `wfdCreateEvent`
  * create an event container with the specified queue size
  * events include a monitor connected/disconnected, 
* `wfdDeviceEventAsync`
  * signal when the event container has something
* `wfdDeviceEventWait`
  * return, or wait, the next event queued
* `wfdDeviceEventFilter`
  * only report the interested events
* `wfdCreatePort`
* `wfdGetPortModes`
  * a mode inlcudes width, height, refresh rate, and etc.
  * they become invalid when the display hardware is re-attached
* `wfdSetPortMode`
* `wfdSetPortAttrib[if][v]`
  * flip or not
  * mirror or not
  * rotation or not
  * power mode: off, suspend, on
  * gamma correction
  * partial refresh
  * protection: encrypted signal
* `wfdBindPipelineToPort`
* `wfdGetDisplayData`
  * return the EDID, for example, data on a port
* `wfdCreatePipeline`
  * there can be no pipeline, if GPU is the display controller.  Pipeline is
    more limited.
  * color space conversion
  * crop
  * flip/mirror/rotate
  * scale/filter
  * layer/blend
* `wfdCreateSourceFromImage`
* `wfdCreateSourceFromStream`
* `wfdCreateMaskFromImage`
* `wfdCreateMaskFromStream`
* `5.6 Using a Pipeline`
  * direct mode: use wfd to control the pipelines
  * indirect mode: use wfc to control the pipelines indirectly
* `wfdBindSourceToPipeline`
* `wfdBindMaskToPipeline`
* `wfdSetPipelineAttrib[if][v]`
  * how a source is sampled and mapped to the output
