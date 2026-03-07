Kernel FAT
==========

## Encodings

- short filenames
  - limited to 8+3 characters
  - encoded in one of the codepages
- long filenames
  - limited to 255 characters
  - encoded in utf-16
- mount options
  - `codepage` is for short filenames
    - default to `CONFIG_FAT_DEFAULT_CODEPAGE` (437)
    - `sbi->nls_disk = load_nls("cp%d")`
    - it converts short filenames between codepage and utf-16, such that short
      filenames can be handled as long filenames
  - `iocharset` is for vfs representation
    - default to `CONFIG_FAT_DEFAULT_IOCHARSET` (`iso8859-1`)
    - `sbi->nls_io = load_nls(sbi->options.iocharset)`
    - it converts filenames between iocharset and utf-16, such that vfs sees
      iocharset instead of utf-16
  - `utf8` hijacks iocharset and utf-16 conversion
    - default to `CONFIG_FAT_DEFAULT_UTF8` (it should be set)
    - it hijacks the conversion such that vfs sees utf-8 instead of iocharset
      - iocharset is still used internally for case-insensitive cmp
- `fat_readdir`
  - `fat_parse_long` reads the utf-16 long filename from the disk
    - `fat16_towchar` is `memcpy` unless endian issue
    - `fat_uni_to_x8` converts utf-16 to the target encoding
      - if `utf8`, it calls `utf16s_to_utf8s` to convert utf-16 to utf-8
      - otherwise, it uses `sbi->nls_io` to convert to `iocharset`
  - `fat_parse_short` reads the short filename from the disk
    - `fat_shortname2uni` uses `sbi->nls_disk` to convert from `codepage` to
      utf-16
    - `fat_uni_to_x8` converts utf-16 to the target encoding

## Physical Format

- <https://en.wikipedia.org/wiki/Design_of_the_FAT_file_system>
- there are 4 regions
  - reserved sectors
    - boot sector
    - fs info sector
    - optional sectors
  - fat region
    - file allocation table #1
    - file allocation table #2 (backup)
  - root directory region (fat12/fat16 only)
  - data region
- boot sector
  - 0x000: jump instruction
  - 0x003: oem name (e.g., `mkfs.fat`)
  - bios parameter block (bpb) below
  - 0x00b: bytes per sector, typically 512
  - 0x00d: sectors per cluster
  - 0x00e: number of reserved sectors, typically 32
  - 0x010: number of file allocation tables, typically 2
  - 0x015: media type, 0xf8 for disk
  - 0x020: total number of sectors
  - 0x024: sectors per file allocation table
  - 0x030: sector of fs info, typically 1
  - 0x032: sector of backup boot sectors, typically 6
  - 0x047: volume label (e.g., `NO NAME`)
  - 0x052: fs type (e.g., `FAT32`)
  - boot code
  - 0x1fe: magic (`0x55 0xaa`)
- fs info sector
  - 0x000: magic (`RRaA`)
  - 0x1e4: magic (`rrAa`)
  - 0x1fc: magic (`0x00 0x00 0x55 0xaa`)
- file allocation table
  - fat consists of an array of entries
  - each entry is 12-/16-/32-bit on FAT12/16/32
  - a dir/file is referred to by a singly-linked list of entries
    - data region is divided into identially-sized clusters
    - each entry points to a cluster in the data region
    - each dir/file occupies one or more clusters

## Mount

- when `do_mount` mounts a vfat fs,
  - `get_fs_type` maps the fstype str to `vfat_fs_type`
  - `fs_context_for_mount` allocs a `fs_context` initialized by
    `vfat_init_fs_context`
  - `fc_mount` creates the superblock by `vfat_get_tree`
- `vfat_init_fs_context` inits fs context
  - `fc->private` inits to `fat_mount_options` with default vals
  - `fc->ops` points to `vfat_context_ops`
- `vfat_get_tree` inits `fc->root`
  - `get_tree_bdev` creates the superblock singleton for the bdev on demand
  - `setup_bdev_super` inits the sb with the bdev info
    - `sb->s_bdev_file` is the opened bdev file
    - `sb->s_bdev` is the bdev
    - `sb->s_bdi` is the bdev info
    - `sb->s_blocksize` is the bdev blocksize
  - `vfat_fill_super` inits the sb with the fs info
    - `sb->s_fs_info` points to `msdos_sb_info`
      - this is the private data for vfat and is shortened to `sbi`
    - `sb->s_op` points to `fat_sops`
    - `setup`
      - `sb->dir_ops` points to `vfat_dir_inode_operations`
      - `sb->__s_d_op` points to `vfat_dentry_ops`
    - `fat_read_bpb` parses bpb from the boot sector
    - `fat_ent_access_init`
      - `sbi->fatent_ops` points to `fat32_ops`
    - `sbi->fat_inode` has ino 0
      - this seems like a dummy inode
    - `sbi->fsinfo_inode` has ino `MSDOS_FSINFO_INO` (2)
    - `root_inode` has ino `MSDOS_ROOT_INO` (1)
    - `fat_read_root` parses the root cluster
      - `inode->i_op` points to `sb->dir_ops`, which is `vfat_dir_inode_operations`
      - `inode->i_fop` points to `fat_dir_operations`
    - `sb->s_root` is `d_make_root(root_inode)`

## Low-Level I/O

- physical entry reading
  - `fatent_init` preps for physical entry access
  - `fat_ent_read` reads the specified entry from FAT
    - `fat_ent_blocknr` translates the entry to bdev block/offset
    - `fat_ent_bread` reads the bdev block
    - `fat32_ent_set_ptr` updates `fatent->u.ent32_p` to point to the entry data
    - `fat32_ent_get` returns the next entry or `FAT_ENT_EOF`
      - this can be used to read the entire list iteratively
  - `fatent_brelse` cleans up
- `fat_get_cluster` caches the entry list for an inode
  - remember that the inode refers to a list of entries in FAT
  - `cluster` is the number of entries to walk or `FAT_ENT_EOF` to walk all
  - `fclus` is the number of entries walked
  - `dclus` is the physical index of the last entry walked in FAT
  - it walks the entries one by one and calls `fat_cache_add` to cache them
- `fat_bmap` maps logical sector of an inode to physical secotr of the bdev
  - `fat_bmap_cluster` maps the logical cluster of the inode to physical
    cluster
    - it calls `fat_get_cluster` to walk the entry list to find the Nth
      physical entry
  - `phys` is the physical bdev sector
  - `mapped_blocks` is the number of contiguous physical sectors that can be accessed
    - that is, `phys` lands in one of the clusters and `phys + mapped_blocks`
      is the end of the cluster

## High-Level I/O

- userspace `readdir` on a directory calls `fat_readdir`
  - `fat_get_entry` reads a physical cluster
    - `fat_bmap` maps the logical inode pos to physical bdev sector
    - `sb_bread` reads the sector
    - because this is a dir, the sector holds a `msdos_dir_entry`
- userspace `read` on a file calls `fat_read_folio`
  - `fat_get_block`
    - `fat_bmap` maps the logical inode pos to physical bdev sector
    - `map_bh` points bh to the physical sector
  - `do_mpage_readpage` has a loop to call `fat_get_block` and
    `mpage_bio_submit_read` submits a bio to read the data from the bdev to
    folio
