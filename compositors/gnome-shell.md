Gnome Shell
===========

## Components

- `src/`
  - `libst.so` provides Shell Toolkit.  It is based on clutter.
  - `libgnome-shell.so` provides Gnome Shell.
  - `libgnome-shell-js.so` provides extension importer.
  - `gnome-shell` uses libmutter and runs as the window manager.  Gnome Shell
    is registered as a plugin of mutter.
  - All libraries are introspected for gjs
- `js/`
  - in `gnome_shell_plugin_start()`, these lines are evaluated
    - `imports.ui.environment.init();`
    - `imports.ui.main.start();`
  - An extension is loaded by `imports.ui.extensionSystem.loadExtension()`
    - it calls `imports.misc.extensionUtils.installImporter()` to set
      property `imports` for the extension
    - then `ext.imports.extension` will load `extension.js` from the extension
      directory
      - gjs resolves `imports` dynamically, and `imports.extension` will load
        `extension.js`
