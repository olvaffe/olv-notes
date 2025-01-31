Chromium Base
=============

## Threads

- <https://chromium.googlesource.com/chromium/src/+/HEAD/docs/threading_and_tasks.md>
- <https://chromium.googlesource.com/chromium/src/+/HEAD/docs/callback.md>
- browser process
  - the main thread ends up in `BrowserMainLoop::RunMainMessageLoop`, running
    `RunLoop`
  - the io thread is created by `BrowserMainLoop::CreateThreads`
  - the thread pool is created by `StartBrowserThreadPool`
- gpu process
  - the main thread ends up in `GpuMain`, running `RunLoop`
  - the io thread is created by `ChildProcess`
  - the thread pool is also created by `ChildProcess`
- task posting
  - a task is essentially a function call wrapped by `base::BindOnce`
  - `base::ThreadPool::CreateSingleThreadTaskRunner` or
    `base::SingleThreadTaskRunner::GetCurrentDefault` returns a single thread
    task runner
    - tasks posted to the single thread task runner are executed in-order on
      the same thread from the thread pool
  - `base::ThreadPool::CreateSequencedTaskRunner` or
    `base::SequencedTaskRunner::GetCurrentDefault` returns a sequenced task
    runner
    - tasks posted to the sequenced task runner are executed in-order, but may
      be on different threads from the thread pool
    - a "sequence" is a virtual thread
  - `base::ThreadPool::PostTask` posts a task to the thread pool
    - two tasks may be executed out-of-order
- physical threads
  - `base::SimpleThread` is an abstract class representing a physical thread
    - `SimpleThread::Start` calls `PlatformThread::CreateWithType` to create a
      physical thread
    - the physical thread runs `base::SimpleThread::ThreadMain` which calls
      `base::SimpleThread::Run` virtual function
  - `base::Thread` is a concrete class representing a physical thread
    - `Thread::Start` calls `PlatformThread::CreateWithType` to create a
      physical thread
    - the physical thread runs `base::Thread::ThreadMain` which enters
      `RunLoop::Run` to handle tasks
