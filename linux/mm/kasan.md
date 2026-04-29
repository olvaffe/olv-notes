# Kernel KASAN

## `CONFIG_KASAN_HW_TAGS`

- concept
  - when a page is allocated, kernel assigns a tag to the page
    - it stores the tag in a memory carveout called MTE
    - it encodes the tag in `page->flags`
  - when the page is freed, kernel clears MTE
    - it stores `KASAN_TAG_INVALID` to MTE
    - it leaves `page->flags` stale
  - upon use-after-free of a page,
    - the hw detects a mismatch
- `page_to_virt` returns the va of a page
  - what we know
    - `vmemmap` is the array of all `struct page`
    - `PAGE_OFFSET` is the va of the first page in the kernel linear map
  - we can find the va using this pseudo code: `PAGE_OFFSET + (page - vmemmap) * PAGE_SIZE`
  - `page_kasan_tag` returns the tag stored in `page->flags`
    - `KASAN_TAG_MIN` to `KASAN_TAG_MAX` are 0xf0 to 0xfd
      - it means the page is allocated and is assigned the given radom tag
    - `KASAN_TAG_KERNEL` is 0xff
      - it means the hw tag is disabled
  - `__tag_set` modifies the va to encode the tag at MSB
- when the cpu accesses the va, the hw compares the tags encoded in the va and
  stored in MTE
