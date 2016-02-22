Android repo
============

## Development

* In Google, there is an internal master branch
  * Long term developments happen on the branch
  * AOSP contributions go to the branch
  * Very unstable
* A named branch is created from master months before a release
  * e.g. `gingerbread`
  * To stabilize
* A -release branch is also created with the named branch
  * e.g. `gingerbread-release`
  * Used to build official releases (e.g., SDK images)
* When the time to open source comes,
  * `gingerbread-release` is filtered to pushed out
  * so does `gingerbread`
* the new public `gingerbread` is merged to public `master`
* With that rationale in mind, take `build/` for example
  * `gingerbread` and `gingerbread-release` were released
  * `gingerbread` was merged to `master`, which was then froyo-based
  * AOSP `master` recevies updates continuously
    * fixes, features for next major release
  * AOSP `gingerbread` receives updates occasionally
    * fixes, features for next minor release
  * AOSP `gingerbread-release` receives updates occasionally
    * just fixes.
    * When it is time to do a minor release, `gingerbread` is
      partially merged.  Features not ready may be omitted.
  * the build id for
    * `master` is `OPENMASTER`
    * `gingerbread` is `GINGERBREAD`
    * `gingerbread-release` is a real id
  * official images are built from `gingerbread-release` (later
    `gingerbread-mr4-release`)
    * release tags are created from this branch
* Which branch to use?
  * current minor release: `android-<version>_<revision>`
  * current minor release plus fixes: `gingerbread-mr4-release`
  * possible next minor release: `gingerbread`
  * bleeding edge: `master`
