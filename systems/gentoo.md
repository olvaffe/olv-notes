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
