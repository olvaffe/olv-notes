## recti

* recti_validate
  * ggl_pick to pick the init_y and scanline
    * init_y will pick step_y and rect
  * recti to set left, right, top.  Then calls rect with the height
* rect_generic
  * step_y to step to the next scanline and then scan
* scanline_t32cb16blend


## pixelflinger

* `shade_t`
  * the values of A, R, G, B, Z, W, and F at pixel (0, 0) and their differentals
    in X and Y directions
* `GGL_ENABLE_SMOOTH` means `glShadeModel(GL_SMOOTH)`
* Pixel pipeline
  * `ggl_pick_texture`
    * check the surface formats of textures and pick the read/write functions
  * `ggl_pick_cb`
    * check the surface format of the color buffer and pick the read/write functions
  * `ggl_pick_scanline`
    * pick `init_y`, `step_y`, and `scanline`
  * `iterators_t` for scanline
    * current `y`
    * left and right bounds: `xl` and `xr`
    * A, R, G, B, Z, W, F values at pixel `(0, y)`
    * `texture_iterators_t` is for a similar purpose
  * `scanline`
* Rasterization
  * 
* how does `shade_t` calculated?
