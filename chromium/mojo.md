Chromium Mojo
=============

## Mojo

- <https://chromium.googlesource.com/chromium/src/+/HEAD/docs/mojo_and_services.md>
- the most common way to create a message pipe between a client and a server
  is
  - the client creates a remote
    - `mojo::Remote<mojom::Foo> foo;`
  - the client creates a pending receiver
    - `mojo::PendingReceiver<mojom::Foo> receiver = foo.BindNewPipeAndPassReceiver();`
  - the client sends the pending receiver to the server
    - e.g., `BrowserInterfaceBroker::GetInterface`
    - `git grep 'GenericPendingReceiver' '*.mojom'`
  - the server provides an implementation
    - `class FooImpl : mojom::Foo { ... };`
    - the ctor converts the pending receiver to a receiver
  - the server registers a handler
    - `mojo::BinderMap::Add<mojom::Foo>(base::BindRepeating(createFoo, ...))`
    - when the server receives the pending receiver, it invokes the handler to
      create an implmentation object
- IO thread
  - browser process
    - `ContentMainRunnerImpl::RunBrowser` calls
      `BrowserTaskExecutor::CreateIOThread` to create the io thread and
      creates a `MojoIpcSupport` to handle mojo messages on the io thread
  - child process
    - `ChildProcess::ChildProcess` creates an `ChildIOThread`
    - `ChildThreadImpl::Init` calls `GetIOTaskRunner` to get the io thread
      task runner and create a `mojo::core::ScopedIPCSupport` to handle mojo
      messages on the io thread
- codegen
  - `FooStubDispatch` class is generated
  - the io thread calls `FooStubDispatch::Accept` with `Foo` and
    `mojo::Message`, and dispatches the message to `FooImpl->SomeMethod`
  - `FooImpl->SomeMethod` either executes the message on the io thread or
    schedules another task to free up io thread
- type maps
  - `BUILD.gn` has `cpp_typemaps` to map between mojo types and internal types
  - e.g., `media/mojo/mojom/BUILD.gn` says `media.mojom.VideoFrame` and
    `::scoped_refptr<::media::VideoFrame>` are convertible
  - the conversion uses `video_frame_mojom_traits.cc`'s `data` and `Read`
    methods

