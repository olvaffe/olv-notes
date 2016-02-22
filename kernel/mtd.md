NAND:
* page: 256 or 512 bytes (or bigger), readable and programmable
* block: 4KB, 8KB or 16KB (or bigger), erasable and programmable
* address and data on the same bus; slow random read; uncapable of random write
* e.g., a 512Mbit nand flash consists of 128 * 1024 512b pages
* the real size of a page is 512+16, where the 16bytes is OOB

Scenerio: we have a 2M NOR mapped in CPU memory address space.

* describe the address/size of the NOR to maps/physmap.c
* physmap creates a map_info and pass it to, say, chips/map_rom.c, which gives mtd_info
* int add_mtd_device(struct mtd_info *mtd) is called to register the mtd_info to the core
* core notifies all notifiers, like mtd_blkdevs and mtdchar.

The controller and storage might be different.  But it can be the same:

* devices/mtdram.c

