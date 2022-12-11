Git
===

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
