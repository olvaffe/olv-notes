# Linux THP

## Transparent Huge Pages (THP)

- it allocates huge pages transparently, unlike hugetlb
- it was for anonymous mappings (e.g., heap and stack) originally
  - it now also supports file mappings and shmem/tmpfs
- on fault, `handle_mm_fault` can call `create_huge_pmd` to allocate hugepage
  for pmd rather than calling `handle_pte_fault` to allocate a regular page
  for pte
- split and collapse
  - split downgrades a hugepage to individual normal pages
    - it involves updating ptes to point to individual pages
    - downgrade enables memory reclaim
  - collapse upgrades individual normal pages to a hugepage
    - `khugepaged` scans for good candidates in the background, and once a
      candidate is found
      - it allocates a hugepage
      - it copies data from individual pages to the hugepage
      - it updates pmd and pte
      - it frees the individual pages
