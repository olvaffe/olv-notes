Git
===

## Branches

- remote branches
  - `git ls-remote --heads <url>` or `git ls-remote --heads <remote>`
  - head references in the remote repo
  - there is an optional default branch, `HEAD`
- remote-tracking branches
  - `git branch -r`
  - local head references that track remote head references
    - stored in `.git/refs/remotes/<name>`
    - updated by `git fetch`
      - `remote.<name>.fetch` is the default refspec
  - to set a default branch locally, `git remote set-head <name> -a`
    - this creates `origin/HEAD` that tracks remote `HEAD`
- local branches
  - `git branch`
  - local head references that may or may not track another branch
    - stored in `.git/refs/heads`
- tracking branches
  - local branches that track other branches
    - branches that are tracked are called upstream branches
    - upstream branches can be any (remote-tracking or local) branches
    - stored in `.git/config`
      - `branch.<name>.remote`
      - `branch.<name>.merge`
  - `git branch --track` or `git branch --set-upstream-to`
    - implied when a branch is created from a remote-tracking branch

## The Object Database

- <user-manual.txt>
- blob object to store file data
  - it is the contents of a file
  - thus, file renames do not create new blob objects
- tree object contains blob object(s) and tree object(s)
  - it can contain commit object(s) when submodule is presented
- commit object contains a tree object and refers to parent commit objects
  - `git show --pretty=raw`
  - `git commit` creates a commit object whose parent is the current HEAD and
    whose tree is taken from the index.
  - The diff is not stored.  It is generated from comparing with the parent's
    tree object.
  - `git ls-tree` to see the contained tree object.
  - `git show <rev>:<path>` to see the tree objects or blob objects
- tag object contains the name and type of another object, and has a symbolic
  name and optionally signature.
  - `git cat-file tag <tag>`
  - a lightweight tag is not a tag object
- packing
  - `git count-objects` to count the number of non-packed objects
  - `git repack` to pack them
  - `git prune` to remove objects that have been packed or unreachable
  - Or, simply `git gc`
  - Objects in a pack are delta compressed.
- A commit changing a single top-level file will give 3 objects
  - A blob object for the file with modified contents
  - A tree object with the new blob object
  - A commit object of the commit itself
  - All objects are deflated.

## The Index

- `.git/index`.
  - To see it, `git ls-files --stage`
- It contains a sorted list of pathes
  - each path has permission, the name of associated blob object, and stage
  - stage is always 0, except for files with merge conflicts.  Each version of a
    conflict file is called a stage, with a different stage number.

## Submodules

- `git submodule add <url>` adds a submodule
  - the submodule repo is cloned to `.git/modules/$SUBMODULE`
  - the submodule repo is checked out at $SUBMODULE
  - a `[submodule]` section is added to `.git/config`
  - a `.gitmodules` file is added
- `git commit` records two files
  - `.gitmodules`
  - `$SUBMODULE` is a special directory for which git only records the commit
    hash
  - when a commit is made to $SUBMODULE locally, `git commit` updates the
    commit hash for the directory
- when one clones a git repo with submodules, one should
  - `git submodule init` to add `[submodule]` to `.git/config`
    - `.gitmodules` serves as a template
  - `git submodule update` clones all submodules and checks out the recorded
    commit hashes

## git config

- `core.pager`
  - use `GIT_PAGER`, then `core.pager`, then `PAGER`, and then `less`
  - if `LESS` is not set, git sets it to `LESS=FRX`

## git gc

- `git gc --auto` is invoked automatically after several other subcommands
  - `gc.autoDetach` is true and runs gc in the background
  - `gc.auto` is 6700.  When there are more loose objects than that number, gc
    packs the loose objects.
  - `gc.autoPackLimit` is 50.  When there are more packs than that number, gc
    consolidates them into one larger pack.

## gerrit and repo

- <https://www.gerritcodereview.com/>
  - <https://gerrit.googlesource.com/gerrit/>
  - <https://gerrit.googlesource.com/git-repo/>
- repo init
  - `repo` is a single-file python script
  - `_FindRepo` finds `.repo/repo/main.py` in the cwd and in the parent dirs
    - if that fails, it prompts for `repo init`
  - `repo init` calls `_Init` 
    - it creates `.repo`
    - `_Clone` and `_Checkout` clones
      `https://gerrit.googlesource.com/git-repo` to `.repo/repo`
    - there is `.repo/repo/main.py` now
  - `repo` invokes `.repo/repo/main.py`
    - `_CheckWrapperVersion` checks the versions of `repo` between the one
      invoked and the one in the cloned repo, and suggests an update if they
      don't match
- repo selfupdate
  - `repo selfupdate` or `repo sync` triggers selfupdate
  - `repoProject` is `.repo/repo`
  - `_PostRepoFetch` prompts for a new repo version and restarts
