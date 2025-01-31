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

## Gjs

- <https://gitlab.gnome.org/GNOME/gjs>
- JavaScript bindingds for GNOME
- soon adopted as the basis of gnome shell's ui code
  - gnome shell is a plugin of mutter
  - mutter was metacity + clutter
  - clutter is deprecated
- gjs defines `imports` for its modules
  - there is `gi` for gobject introspection repository
  - then there are those existed in
    - gjs client (such as gnome-shell) defined pathes
    - `$GJS_PATH`
    - `$XDG_DATA_DIRS /gjs-1.0`
    - `/usr/lib/gjs-1.0`
    - `/usr/share/gjs-1.0`
- `imports` is defined by `gjs_create_root_importer()`
  - its properties are resolved at runtime by `importer_new_resolve()`
  - when the interpreter sees `imports.foo`,
    - it checks if `foo` is an internal module
    - if not, it checks if `foo.js` exists.  If it does, it constructs a
      new object and `foo.js` is evaluted using the new object as the global
      object.  `imports.foo` is defined to be the new object.
    - if not, it checks if `foo.so` exists.  `foo.so` is loaded similarly
