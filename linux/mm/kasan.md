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
