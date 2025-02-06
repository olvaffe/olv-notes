rsync
=====

## Overview

- <https://rsync.samba.org/>
- <https://rsync.samba.org/how-rsync-works.html>
  - Process Startup
    - client rsync forks ssh to connect to server and to start server rsync
      - client rsync is sender
      - server rsync is receiver
      - they use the ssh tunnel to communicate
    - sender and receiver negotiate the rsync protocol version
    - sender sends exclude list
  - The File List
    - sender builds the file list and sends it to receiver
      - the file list includes pathnames, ownerships, mode, permissions, size,
        modtime, and if `--checksum`, checksums as well
    - receiver forks to become generator/receiver pair
  - The Pipeline
    - generator -> sender -> receiver
  - The Generator
    - generator checks local sizes/modtimes against those in the file list,
      and skips those that match
      - if `--checksum`, generator checks checksums instead
    - if a file is not skipped, generator asks sender to send the file
      - if the file exists, generaor asks sender to send delta
  - The Sender
    - sender calculates the delta which is the heart of the rsync algorithm
    - sender sends delta to receiver
  - The Receiver
    - receiver rebuilds each file in a temp file, fixes owner, mode, perms,
      and renames the temp file to the final name
- experiment
  - there are two processes on local
    - `rsync`
      - stdio connects to the controlling tty
      - there are two sockets connecting to ssh
        - it forks, makes one socket stdin and one socket stdout, and execves
          ssh
    - `ssh <remote> rsync --server`
      - there is a socket connecting to remote
      - there are two sockets (for in and out respectively) connecting to rsync
      - it forwards data between remote and rsync
  - there are four processes on remote
    - `sshd: olv [priv]` waits for the session to end to cleanup
    - `sshd: olv@notty`
      - there is a socket connecting to local
      - there are 3 pipes connecting to `rsync --server`
      - it forwards data between the pipes and the socket
    - `rsync --server` x2
      - generator and receiver

## Common Options

- `-a` is equivalent to `-rlptgoD`
  - `-r` recurses into dirs
  - `-l` copies links as links
  - `-p` preserves permissions
  - `-t` preserves mtimes
    - we always want this because rsync checks both size and mtime to skip
  - `-g` preserves groups
  - `-o` preserves users
  - `-D` preserves special files and device files
