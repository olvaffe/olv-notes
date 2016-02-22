GObject Introspection
=====================

## Overview

* `g-ir-scanner` scans the source files to create GIR XML format
  * other than scanning the source files, it also build an executable to dump
    GTypes of the library.  The binary can dump the properties and signals of
    the library's GTypes.
  * run `g-ir-scanner` with `GI_SCANNER_DEBUG=save-temps` to see the source
    file of the executable files
* `g-ir-compiler` compiles GIR XML files into typelibs
* `libgirepository` is a C library for accessing typelib

## Support GObject Introspection

* add `GOBJECT_INTROSPECTION_CHECK` to configure.ac
* add these to Makefile.am
  * `include $(INTROSPECTION_MAKEFILE)`
    * `/usr/share/gobject-introspection-1.0/Makefile.introspection`
  * `INTROSPECTION_GIRS = List of GIRs to be generated`
  * `INTROSPECTION_SCANNER_ARGS = Additional args for g-ir-scanner`
  * `INTROSPECTION_COMPILER_ARGS = Additional args for g-ir-compiler`
* 
