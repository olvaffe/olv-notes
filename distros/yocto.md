Yocto Project
=============

## Overview

- tools to create embedded linux distributions
- poky
  - a reference distribution
  - can be used to create other distributions
- I guess it is like how cros uses gentoo to create a chroot to create other
  gentoo-based images

## BitBake

- http://olvaffe.blogspot.com/2008/12/bitbake-data-class.html

handle and feeder functions of bb/parse/parse_py/BBHandler.py:

A = "3" -> setVar('A', '3') -> setVarFlags('A', 'content', '3')
A += "4" -> setVar('A', '3 4') -> setVarFlags('A', 'content', '3 4')
A_append = "5" -> setVar('A_append', '5') -> setVarFlags('A', 'append', ['5'])
A_append = "6" -> setVar('A_append', '6') -> setVarFlags('A', 'append', ['5', '6'])
A[dirs] = "${D}" -> setVarFlags('A', 'dirs', '${D}')

A_${PN} = "7" -> setVar('A_${PN}', '7') -> setVarFlags('A_${PN}', 'content', '7'), seen override but not used
					-> usually followed by expandKey, and ${PN} is expanded


Two classes, A, B.  A has nothing.  B inherits A.
In B, EXPORT_FUNCTIONS do_package ->
do_package calls B_do_package, which calls A_do_pacakge


bitbake -e 

-> bb.parse.handle(bitbake.conf)
conf/bitbake.conf
conf/site.conf
conf/auto.conf
conf/local.conf
conf/build/i686-linux.conf
conf/target/INVALID-INVALID.conf
conf/machine/qemuarm.conf
conf/machine/include/qemu.inc
conf/machine/include/tune-arm926ejs.inc
conf/machine/include/tune-thumb.inc
conf/distro/openmoko.conf
conf/distro/include/preferred-om-2008-versions.inc
conf/distro/include/angstrom-2007-for-openmoko.inc
conf/distro/include/sane-srcdates.inc
conf/distro/include/sane-srcrevs.inc
conf/distro/include/preferred-xorg-versions.inc
conf/distro/include/preferred-gpe-versions-2.8.inc
conf/distro/include/preferred-e-versions.inc
conf/distro/include/angstrom.inc
conf/distro/include/angstrom-glibc.inc
conf/distro/include/angstrom-package-ipk.inc
conf/documentation.conf
conf/sanity.conf
conf/abi_version.conf
conf/enterprise.conf
-> parse 'base' class and those listed in INHERITS
classes/base.bbclass
classes/patch.bbclass
classes/siteinfo.bbclass
classes/devshell.bbclass
classes/src_distribute_local.bbclass
classes/src_distribute.bbclass
classes/debian.bbclass
classes/sanity.bbclass
classes/devshell.bbclass
classes/angstrom-mirrors.bbclass
classes/insane.bbclass
classes/package.bbclass
classes/testlab.bbclass
classes/package_ipk.bbclass
classes/packaged-staging.bbclass
classes/sanity.bbclass

