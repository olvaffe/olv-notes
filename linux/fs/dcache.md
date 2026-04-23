Linux dentry cache
==================

## Overview

- we want to resolve `/a/b/c` to an inode
  - `/` inode is known
  - we look up `/a` inode under `/` inode
  - we look up `/a/b` inode under `/a` inode
  - we look up `/a/b/c` inode under `/a/b` inode
- that is the slowest path of `link_path_walk`
  - `walk_component` calls `lookup_slow` to look up next component
  - `inode->i_op->lookup`
    - fs searches under the dir inode for the component
    - fs allocs a inode for the component on demand
    - fs calls `d_splice_alias`
      - `dentry->d_inode` is set to `inode` of the component
- dentry and inode
  - a dentry without an inode is a negative dentry: path does not exist
  - an inode without a dentry:
    - pipe, socket, kernel internal, etc.
    - deleted but open
    - dentry evicted by reclaim
  - multiple dentries share an inode: hard links
    - these dentries are known as alises
    - aliases are valid for files but invalid for dirs
    - `d_splice_alias` avoids dir aliases

## dentry lifetime

- `d_alloc` allocs a dentry with the specified name
  - `dentry->d_op->d_init`, which is very rare
- `dput` calls `__dentry_kill` if it decides to destroy
  - `dentry->d_op->d_prune`, which is very rare
  - `dentry->d_op->d_iput`, which is very rare
    - it falls back to `iput`
  - `dentry->d_op->d_release`, which is very rare
- how `fput` leads to `d_put`
  - `fput -> __fput`
    - `file->f_op->release`
    - `dput`
