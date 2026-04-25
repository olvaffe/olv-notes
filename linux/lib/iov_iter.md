# Kernel `iov_iter`

## Overview

- an `iov_iter` can wrap many iterators
  - `iov_iter_ubuf` is for userspace buf with `ITER_UBUF`
  - `iov_iter_init` is for `iovec` with `ITER_IOVEC`
  - `iov_iter_bvec` is for `bio_vec` with `ITER_BVEC`
  - `iov_iter_kvec` is for `kvec` with `ITER_KVEC`
  - `iov_iter_folio_queue` is for `folio_queue` with `ITER_FOLIOQ`
  - `iov_iter_xarray` is for `xarray` with `ITER_XARRAY`
  - `iov_iter_discard` is null with `ITER_DISCARD`
