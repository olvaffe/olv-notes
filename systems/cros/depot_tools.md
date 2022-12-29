Chrome OS `depot_tools`
=======================

## Overview

- <https://chromium.googlesource.com/chromium/tools/depot_tools.git>
- running a command can sliently `git pull` the repo
- `fetch` fetches pre-defined projects using `gclient`
- `gclient` is a meta-checkout tool
  - it parses `.gclient` which is set up by `fetch`
  - it also parses `DEPS` which is checked in to the project
- `git cl` interacts with gerrit
