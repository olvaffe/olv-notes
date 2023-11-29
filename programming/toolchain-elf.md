ELF
===

## History

- `a.out` was used in the first edition of UNIX
- `COFF` superceded `a.out` in SysV 1 in 1983
  - `PE/COFF` is still used on windows and uefi
  - there is also `Mach-O` used on macos/ios
- `ELF` superceded `COFF` in SysV 5

## ELF

- An ELF file starts with ELF header at file offset 0
  - `e_type`, `ET_DYN` for relocatable and `ET_EXEC` for fixed-address
  - `e_entry`, where the kernel should jump to, or 0
  - `e_phoff`, where the program headers are in the file, or 0
  - `e_shoff`, where the section headers are in the file, or 0
  - `e_ehsize`, ELF header size
  - `e_phentsize` and `e_phnum`, program header size and count
  - `e_shentsize` and `e_shnum`, section header size and count
- A program header describes a segment.  For `PT_LOAD`,
  - `p_offset` is the file offset of the first byte of the segment
  - `p_filesz` is the size of the file image of the segment
  - `p_vaddr` is the virtual address of the segment
  - `p_memsz` is the size of the memory image of the segment
  - `p_flags` is R, W, X
  - `p_align` must be the page size
- A `PT_INTERP` program header specifies the dynamic loader
  - on x86-64, normally the dynamic loader and the executable are both
    relocable.  The kernel loads both at random addresses.
- A section descrbies a section
  - `sh_name`, the name of the section
  - `sh_type`, the type of the section
  - `sh_offset`, the offset of the section in the file
  - `sh_size`, the size of the section in the file
  - `sh_addr`, the address of the section in the memory
- one ELF header followed by
  Program header table, describing zero or more segments
  Section header table, describing zero or more sections
  Data referred to by entries in the program header table, or the section header table
- sections are non-overlapping.  some bytes might not be covered by any section.
- normally, a segment contains one or more sections
- segments are used by kernel for mapping and execution, while sections
  contains data for relocation and linking
- segments of type `PT_LOAD` are to be mapped.  memory size is greater than or
  equal to file size (e.g. bss has no file size)
- e.g. to map two segments
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000000 0x08048000 0x08048000 0x4e124 0x4e124 R E 0x1000
  LOAD           0x04e124 0x08097124 0x08097124 0x00730 0x00aa0 RW  0x1000
  there will be two vma (0x08048000 ~ 0x08097000, size 0x4f000)
                        (0x08097000 ~ 0x08098000, size 0x01000)
  a portion of the file is mapped twice.
