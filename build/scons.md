SCons
=====

## Command line options

* `scons` builds the project.
* `scons -c` cleans up the build.
* `scons -Q` supresses SCons status messages

## Scripts

* `SConstruct` is `Makefile` equivalent
* `SConscript`
* By default, SCons uses `Decider('MD5')`.  It uses MD5 to detect if a source
  has changed and if a rebuild is required.
  * It uses a database to track the states

## Builder Methods

* `Program('<src>')` builds a program from the sources
  * Or `Program(['<src1>', '<src2>'])` to build from multiple sources
  * Or `Program('<app-name>', '<src>')` to set the program's name
* `Library` or `StaticLibrary` builds a static library.  `SharedLibrary` builds
  a shared library.  They have a syntax similiar to `Program`.
  * Other than sources, you can list objects directly
* The above builder methods also accept keyword arguments
  * `target` for target name
  * `source` for source list
  * `LIBS` for the libraries to link to
  * `LIBPATH` for the additional directories to search for libraries
  * `CPPPATH` for the additional include directories.  It is better than
    `CCFLAGS` in that
    * it is portable
    * it helps scons derive the implicit dependencies (source A depends on
      header B in the listed directory).
    * it understands `#/include`, which means the `include` directory of the
      top directory.
  * `CCFLAGS` for `CFLAGS`
* `Object('<src>')` builds an object from the sources

## Node Objects

* every builder method will return a node object lists.  The list can in turn be
  used as a source list.
  * `objects = Object('a.c', CCFLAGS='-DGOOD')`
  * `Program('prog', objects)`
* Actually, every source will be converted to a node in SCons.  This can also be
  done explicitly
  * `File('a.c')` returns a file node
  * `Dir('a')` returns a directory node

## Dependencies

* There are implicit dependencies.
* There are also explicit dependencies
  * `hello = Program('hello.c')`
  * `Depends(hello, 'other_file')`
* To summarize, there are
  * `Depends`
  * `ParseDepends` to parse a Makefile-like dependency
  * `Ignore` explictly removes a file from dependencies
  * `Requires` means order-only dependency
  * `AlwaysBuilds`

## Construction Environment

* External Environment: environment variables
  * enabled by `import os` in the script
* Construction Environment:
  * created by `Environment`
* Execution Environment:
  * is a sanitized shell environment, not the same as the external environment
  * can be changed by changing `ENV` variable of the construction environment
  * there is simple function to manipulate it
    * `env.PrependENVPath('PATH', '/usr/local/bin')`
* All builders changes `DefaultEnvironment` when no environemnt is given
* A construction environment is maniuplated by
  * `env[key]` returns the value of key
  * `env.subst($key)` return the value of key, expand everything recursively
  * `Clone` clones the environment
  * `Replace` replaces a variable
  * `SetDefault` sets a variable if not exists
  * `Append` and `AppendUnique`
  * `Prepend` and `PrependUnique`
* GCC and pkg-config
  * `ParseFlags` to parse a gcc-style commandline into a dictionary with
    keys such as `CPPPATH` or `LIBPATH`; `MergeFlags` merges the dictionary.
  * Even simpler, one can `ParseConfig("pkg-config --cflags --libs x11")`

## Control SCons from command line (ch. 12)

* `Options`
* `Variables`
* `Targets`

## Hierarchical Builds

* `SConscript`
* Export python variables to scripts
  * `SConscript('src/SConscript', exports='<var-name>')`
  * Or globally, `Export('<var-name>')`
* To import, `Import('<var-name>')` or `Import('*')`
