# Kernel KASAN

## `CONFIG_KASAN_HW_TAGS`

- armv8.5
  - MTE: memory carveout to store 4-bit tag for each 16-byte region
  - TBI: ignore the top byte of va
- slab allocator
  - when `kmalloc` allocates, it generates a random tag, encodes the tag in
    va, and stores the tag in mte
    - `kasan_slab_alloc`
      - `assign_tag` generates a tag
      - `set_tag` encodes the tag in va
      - `unpoison_slab_object` stores the tag in mte
  - when `kfree` frees, it poisons mte
    - `kasan_slab_free` calls `poison_slab_object` to poison mte to
      `KASAN_SLAB_FREE`
  - upon use-after-free, mmu detects the tag mismatch and raises an exception
    - the tag encoded in the va is called pointer tag
    - the tag stored in the mte is called memory tag
- buddy allocator is similar
  - when `alloc_pages` allocates, `kasan_unpoison_pages`
    - `kasan_random_tag` generates a tag
    - `kasan_unpoison` stores the tag in mte
    - `page_kasan_tag_set` encodes the tag in `page->flags`
  - when `__free_pages` frees, `kasan_poison_pages` poisons mte to
    `KASAN_PAGE_FREE`
  - `page_to_virt` returns the tagged va of a page
    - what we know
      - `vmemmap` is the array of all `struct page`
      - `PAGE_OFFSET` is the va of the first page in the kernel linear map
    - we can find the real va: `PAGE_OFFSET + (page - vmemmap) * PAGE_SIZE`
    - `page_kasan_tag` returns the tag stored in `page->flags`
    - `__tag_set` encodes the tag in va
- when slab allocator calls `allocate_slab` to allocate from buddy allocator,
  - because it is a buddy allocation, a tag is generated and stored/encoded in
    both mte and `page->flags`
  - `kasan_poison_slab` then calls
    - `page_kasan_tag_reset` to reset `page->flags` to `KASAN_TAG_KERNEL`
    - `kasan_poison` stores `KASAN_SLAB_REDZONE` to mte
- vmalloc allocator
  - when `vmap` maps, `kasan_unpoison_vmalloc` is called with `KASAN_VMALLOC_PROT_NORMAL`
    - it is nop and there is no tagging
    - it relies on buddy allocator page tagging instead
  - when `vunmap` unmap, `kasan_poison_vmalloc`
    - it is also nop
  - when `vmalloc` allocates,
    - the pages are allocated with `__GFP_SKIP_KASAN` and `__GFP_SKIP_ZERO`
      - they skip buddy allocator page tagging
    - `kasan_unpoison_vmalloc` is called with `KASAN_VMALLOC_PROT_NORMAL`,
      `KASAN_VMALLOC_VM_ALLOC`, and potentially `KASAN_VMALLOC_INIT`
      - `kasan_random_tag` generates a tag
      - `set_tag` encodes the tag in va
      - `kasan_unpoison` stores the tag in mte
      - if the alloc size is not page-aligned, `kasan_poison` stores
        `KASAN_TAG_INVALID` in mte for the redzone
      - `unpoison_vmalloc_pages` calls `page_kasan_tag_set` to store tag in
        `page->flags`
        - this is for `vmalloc_to_page`
