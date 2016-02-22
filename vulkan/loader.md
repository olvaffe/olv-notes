Vulkan Loader
=============

## Loader-Layer interface

* instance layers must export `vkEnumerateInstanceLayerProperties` and
  `vkEnumerateInstanceExtensionProperties`
* device layers must export `vkEnumerateDeviceLayerProperties` and
  `vkEnumerateDeviceExtensionProperties`
