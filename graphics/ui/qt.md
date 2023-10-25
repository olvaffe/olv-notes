Qt
==

## `QApplication`

- class hierarchy
  - `QCoreApplication` is for non-gui apps
    - it contains the main event loop where events from the os and other
      sources, such as timers or network, are processed and dispatched
  - `QGuiApplication` is for gui apps
    - events from the window system are also processed and dispatched
    - it also handles most of the system-wide and app-wide settings
    - command line options
      - `-platform` specifies the semicolon-separated QPA (Qt Platform
        Abstraction) plugins
        - such as `android`, `cocoa`, `ios`, `offscreen`, `windows`,
          `wayland`, and `xcb`
        - env `QT_QPA_PLATFORM`
        - some apps static-link to qt and only support the `xcb` platform
      - `-platformpluginpath` specifies the path to plugins
        - env `QT_QPA_PLATFORM_PLUGIN_PATH`
      - `-platformtheme` specifies the theme
        - env `QT_QPA_PLATFORMTHEME`
      - `-plugin` specifies additional plugins to load
        - env `QT_QPA_GENERIC_PLUGINS`
      - `-qwindowgeometry WxH+X+Y` specifies the geometry
  - `QApplication` is for `QWidget`-based gui apps
    - command line options
      - `-style` sets the style
        - env `QT_STYLE_OVERRIDE`
      - `-stylesheet` sets the style sheet
