Kernel VHOST
============

## vhost

- in-kernel virtio device implementation for host kernel
  - it is partial implementation that only processes virtqueue buffers
- to use a vhost driver, the userspace should
  - create a virtio device as it normally would
  - open `/dev/vhost-blah`
  - `VHOST_SET_OWNER`
    - the vhost driver spawns a `vhost-<pid>` thread
  - `VHOST_GET_FEATURES`
    - get the virtio device features the vhost driver supports
  - `VHOST_SET_FEATURES`
    - tell the vhost driver the negotiated features
  - `VHOST_SET_MEM_TABLE`
    - tell the vhost driver the hva-gpa mappings
  - for each virtqueue
    - set up an eventfd pair for sending irqs
      - give the write end to the vhost driver using `VHOST_SET_VRING_CALL`
      - give the read end to KVM
    - set up an eventfd pair for receiving notifications
      - give the write end to KVM
      - give the read end to the vhost driver using `VHOST_SET_VRING_KICK`
    - `VHOST_SET_VRING_x` to describe the vring to the vhost driver
- the vhost driver creates a thread `vhost-<pid>` to drive the virtqueues
  - when the guest driver kicks, the thread is woken up to process the guest
    buffers
  - when it has buffers for the guest, the thread is woken tup to send the
    buffers

## vhost-user

- it is similiar to vhost except the device is implemented by a
  process separated from the hypervisor
- the transfort is a UNIX socket rather than a misc dev `/dev/vhost-blah`
- the protocol mirrors the vhost ioctls
  - the main issue is the hva-gpa mapping
  - the guest physical memory should be backed by a memfd in host, such that
    it can be sent to the other process
- no kernel support needed

## virtio-vhost-user (experimental and appears dead)

- this is a virtio device
- it provides a new transport method, virtio-vhost-user to replace UNIX socket
  in vhost-user
- this allows the device to be implemented by a process in another VM
