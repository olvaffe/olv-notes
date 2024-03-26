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
