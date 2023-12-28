Linux pipe
==========

## Pipes

- pipe(7)
- pipe(3) returns two fds; first for reading and second for writing
- this enables one-way communication
- `fs/pipe.c`
  - pipe(2) creates a `S_IFIFO` inode in an internal pipefs fs
  - the inode's `i_fop` is set to `pipefifo_fops`
  - the inode is opened twice,  `O_RDONLY` and `O_WRONLY` respectively

## FIFOs

- fifo(7)
- mkfifo creates a special (`S_IFIFO`) file that can be opened to create
  reading end and writing end
- there can be multiple readers and multiple writers
  - semantics unclear
- work exactly like pipe internally

