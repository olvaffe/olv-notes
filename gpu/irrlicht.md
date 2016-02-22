Irrlicht
========

## Namespaces

* `irr`
* `irr::core`
* `irr::scene`
* `irr::video`
* `irr::io`
* `irr::gui`

## `createDevice`

* It mainly does two things
  * `createWindow` if the driver is not `video::EDT_NULL`.  It chooses a
    visual, creates a window and a context, and make them current
  * `createDriver`
