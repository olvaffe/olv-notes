Linux kthread
=============

## Worker

- `kthread_create_worker` or `kthread_run_worker` creates a kthread worker
  - `kthread_worker_fn` is the thread entrypoint
    - it pops and execs `worker->work_list` in order
- `kthread_queue_work` queues a work
  - it adds the work to `worker->work_list` and wakes up the worker
