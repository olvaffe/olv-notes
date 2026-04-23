DRM File
========

## Overview

- `drm_open` is the open fop
  - `filp->f_mapping` points to `dev->anon_inode->i_mapping`
  - `drm_file_alloc` allocs a `drm_file`
    - `file->client_id` is monotonically-increasing unique id
    - if root, `file->authenticated` is true
    - `drm_gem_open` inits `file->object_idr`
    - `drm_syncobj_open` inits `file->syncobj_xa`
    - `drm_prime_init_file_private` inits `file->prime`
    - `dev->driver->open` inits `file->driver_priv`
  - if primary node, `drm_master_open` makes the first client master
  - the file is added to `dev->filelist`
- `drm_release` is the release fop
  - the file is deleted from `dev->filelist`
  - `drm_file_free` frees the file
    - `drm_events_release` frees unconsumed events
    - `drm_fb_release` frees `file->fbs`
    - `drm_property_destroy_user_blobs` frees `file->blobs`
    - `drm_syncobj_release` frees `file->syncobj_xa`
    - `drm_gem_release` frees `file->object_idr`
    - if primary node, `drm_master_release`
    - `dev->driver->postclose` frees `file->driver_priv`
    - `drm_prime_destroy_file_private` frees `file->prime`
  - if this is the last file of a dev, `drm_lastclose` calls
    `drm_client_dev_restore` to restore the in-kernel client (e.g., fbdev emu)

## Events

- `drm_send_event` sends an event to userspace
  - e.g., vblanks
  - it completes `e->completion` which points to `commit->flip_done`
  - it signals `e->fence` which points to out-fence
  - it adds the event to `file_priv->event_list`
  - it wakes up `file_priv->event_wait`
- `drm_read`
  - it reads events from `file_priv->event_list` to the user buffer
- `drm_poll`
  - it polls `file_priv->event_list` with `file_priv->event_wait`

## fdinfo

- `drm_show_fdinfo`
  - `drm-driver` is driver name
  - `drm-client-id` is unique client id
  - if pci, `drm-pdev` is pci slot
  - if `DRM_IOCTL_SET_CLIENT_NAME`, `drm-client-name` is client nmae
  - `dev->driver->show_fdinfo` prints driver-specific fdinfo
- `drm_show_memory_stats` helper
  - it collects `drm_memory_stats` from all gem objs (`file->object_idr`)
  - `drm_print_memory_stats` prints
    - `drm-total-memory` is size of private plus shared objs
    - `drm-shared-memory` is size of shared objs
    - `drm-active-memory` is size of shared objs
    - `drm-resident-memory` is size of resident objs
    - `drm-purgeable-memory` is size of purgeable objs
