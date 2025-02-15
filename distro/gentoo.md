Gentoo
======

## Portage

- man 5 portage
  - /etc/make.conf and /etc/portage/make.conf
  - PORTDIR="/usr/local/portage/stable" for path to the main repository
  - `PORTDIR_OVERLAY` for additional repositories
  - DISTDIR="/var/lib/portage/distfiles" for path to store downloaded sources
  - `PORT_LOGDIR="/var/log/portage` for log files
  - PKGDIR="/var/lib/portage/pkgs" where to store built packages
- an overlay is an ebuild repo
  - the main repo specified by PORTDIR is also an overlay
  - metadata/layout.conf
    - masters specifies other repos that this overlay can use and depend on
  - profiles/base/make.defaults
    - make.conf-like configs for the overlay
- <https://dev.gentoo.org/~zmedico/portage/doc/man/make.conf.5.html>
  - `FEATURES`
    - for features of portage itself
  - `USE`
    - for optional features of packages and their dependencies
- <https://dev.gentoo.org/~zmedico/portage/doc/man/portage.5.html>
  - `package.mask`
    - mask out packages from being installed
  - `package.unmask`
    - unmask packages
  - `package.use`
    - set/unset USE flags for packages
  - `package.use.mask`
    - mask out or unmask USE flags for packages
  - `use.mask`
    - mask out or unmask USE flags globally
- `emerge`
- `equery`

## Gentoo Development Guide

- <https://devmanual.gentoo.org/>
- Quickstart ebuild guide
- General concepts
- Ebuild writing
  - `<name>-<version>.ebuild`
    - non-normal releases (alpha, beta, rc, patch, etc.) may be suffixed
    - gentoo revisions may be sufficed if any, starting from `-r1`
    - live ebuilds use 9999 as the version
  - use tabs for indention, with `tabstop=4`
  - `EAPI=n` without quotation
    - use 7 or later
  - `if use <foo>` or `use <foo> && ...`
    - `if ! use <foo>` or `use <foo> || ...` for negation
  - `die` for fatal errors
  - `einfo` (green, not logged), `elog` (green), `ewarn` (yellow), `eerror` (red)
  - Predefined read-only variables
    - `P` is `<name>-<version>`
    - `PN` is `<name>`
    - `PV` is `<version>`
    - `PR` is gentoo revision
    - `PF` is `<name>-<version>-<revision>`
    - `WORKDIR` is path to build dir, such as `${PORTAGE_BUILDDIR}/work`
    - `T` is path to temp dir, such as `${PORTAGE_BUILDDIR}/temp`
    - `D` is path to temp install dir, such as `${PORTAGE_BUILDDIR}/image`
  - Ebuild-defined variables
    - `EAPI` is the eapi
    - `SRC_URI` is a list of source uris
    - `SLOT` is the slot; use `SLOT="0"` if not needed
    - `KEYWORDS`
    - `IUSE` is a list of USE flags
    - `REQUIRED_USE` validates USE combos
    - `DEPEND` is for `CHOST`
    - `BDEPEND` is for `CBUILD`
    - `RDEPEND` is for runtime deps
    - `S` defaults to `${WORKDIR}/${P}`
  - User environment
    - `AR` and `ARFLAGS`
    - `AS` and `ASFLAGS`
    - `CC`, `CFLAGS`, and `CPPFLAGS`
    - `CXX` and `CXXFLAGS`
    - `LD` and `LDFLAGS`
  - `inherit <eclasses>`
  - Ebuild phase functions
    - `pkg_pretend` for early sanity checks during dep calculation
    - `pkg_setup` for early checks and configs before building
    - `src_unpack` unpacks package, default to unpack all `SRC_URI`
    - `src_prepare` prepares package, default to apply all `PATCHES`
    - `src_configure` configures package
      - `econf` uses
        - `--prefix="${EPREFIX}"/usr`
        - `--build="${CBUILD}"`
        - `--host="${CHOST}"`
        - `--target="${CTARGET}"`
        - `--with-sysroot="${ESYSROOT:-/}"`
    - `src_compile` builds package
    - `src_test` runs pre-install tests
    - `src_install` installs package to `D`
    - `pkg_preinst` is called before installing image to `ROOT`
    - `pkg_postinst` is called after installing image to `ROOT`
  - `metadata.xml` specifies metadata about a package
- Ebuild maintenance
- Eclass writing guide
  - `@ECLASS_VARIABLE` is a global variable
  - `@FUNCTION` is a function
  - `@VARIABLE` is a function variable
- Profiles
  - `profiles/categories`
  - `profiles/info_*`
  - `profiles/*/make.defaults`
  - `profiles/*/package.mask`
  - `profiles/*/packages`
  - `profiles/updates/`
  - `profiles/use.*.desc`
  - `profiles/*/use.mask`
- Keywording and stabilization
  - `KEYWORDS`
    - `<arch>`: stable
      - e.g., `amd64`, `x86`, `arm64`, `arm`, `sparc`
    - `~<arch>`: testing
    - `-<arch>`: known bad
    - `*`: wildcard
      - `-* <arch>`: only stable on `<arch>`
      - do not use `*` or `~*`
        - cros uses `~*` for 9999 ebuilds
- Tasks reference
- Function reference
- Eclass reference
  - `flag-o-matic.eclass` provides functions to work with compiler flags
  - `python-any-r1.eclass` determines best python version based on
    `PYTHON_COMPAT`
- Tools reference
- Hosted projects
- Arch specific notes
- Appendices

## Dependencies

- <https://devmanual.gentoo.org/general-concepts/dependencies/>
- cross-compiling
  - `CBUILD` is the system that the package is built
  - `CHOST` is the system that the package is executed
- dependencies
  - `BDEPEND` is build dependencies on `CBUILD` (e.g., `pkgconfig`, `python`)
    - this is added in EAPI 7
    - without this, for example, if `python` was listed in `DEPEND`, `python`
      would need to be built for both `CBUILD` and `CHOST`
  - `DEPEND` is build dependencies on `CHOST` (e.g., linked libraries)
  - `RDEPEND` is runtime dependencies (e.g., dynamically-linked libraries,
    data files)
    - a dependency common to `DEPEND` and `RDEPEND` must be listed in both
      places
- syntax
  - basic: `foo-category/bar-name` such as `dev-lang/ruby`
  - version: `>=foo-category/bar-name-version` such as
    `>=dev-libs/openssl-0.9.7d`
    - supported specifiers: `>=`, `>`, `~`, `=`, `<=`, `<`
    - use `~` instead of `=`: `~` ignores revisions
  - ranged: `=foo-category-bar-name-ver*` such as `=x11-libs/gtk+-2*`
  - blocker: `!foo-category/bar-name`
    - supported specifiers: `!`, `!!`, `!<`, and other verion specified
      - `!` installs the package before removing the blocker
      - `!!` installs the package after removing the blocker
  - slot: `foo-category/bar-name:slot` such as `dev-qt/qtcore:5`
    - slots are for packages that can have multiple verions in parallel such
      as gtk 2/3/4 or qt 4/5/6
    - `:=` means any slot is fine, but the package is rebuilt when the
      dpenendency changes slot or sub-slot (e.g., gtk 2 to 3)
    - `:*` means any slot is fine, and no rebuild when the dependency changes
      slot
    - `:SLOT=` means only SLOT is fine, and the package is rebuilt when the
      dependency changes sub-slot
    - `:SLOT` means only SLOT is fine, and no rebuild when the dependency
      changes sub-slot
    - `:SLOT/SUBSLOT` means only SLOT/SUB-SLOT is fine
  - USE-conditional: `baz? ( foo-category/bar-name)`
    - `!baz` for negation
    - nesting is fine
  - Any: `|| ( app-misc/foo app-misc/bar )"`
  - require USE: `app-misc/foo[-bar,baz]`
    - `foo` must have `baz` and must not have `bar`
    - `app-misc/foo[bar?]` is shorthand for `bar? ( app-misc/foo[bar] ) !bar? ( app-misc/foo )`

## USE Flags

- <https://devmanual.gentoo.org/general-concepts/use-flags/index.html>
- `IUSE` defaults
  - `+`/`-` enables/disables a USE flag by default

## Package Manager Specification

- <https://wiki.gentoo.org/wiki/Project:Package_Manager_Specification>
  - standarized ebuild environment
  - versions
    - EAPI 0: 2008, portage 2.0.53
    - EAPI 1: 2008, portage 2.1.3.19
    - EAPI 2: 2008, portage 2.1.6.4
    - EAPI 3: 2010, portage 2.1.7.17
    - EAPI 4: 2011, portage 2.1.9.42
    - EAPI 5: 2012, portage 2.1.11.31
    - EAPI 6: 2015, portage 2.2.26
    - EAPI 7: 2018, portage 2.3.40-r1
      - cros uses EAPI 7
    - EAPI 8: 2021, portage 3.0.20-r6
