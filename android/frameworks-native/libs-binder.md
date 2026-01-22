Android Binder
==============

## IPC Overview

- native client
  - it gets hold of a `BpBinder`
  - it calls `BpBinder::transact` to send a transaction
    - `IPCThreadState::transact` batches the `BC_TRANSACTION` transaction in `mOut`
    - `IPCThreadState::flushCommands` calls `IPCThreadState::talkWithDriver`
      to flush `mOut` to kernel binder
- native server
  - the main thread is in `IPCThreadState::joinThreadPool` loop
  - `IPCThreadState::getAndExecuteCommand`
    - `IPCThreadState::talkWithDriver` reads the `BR_TRANSACTION` and batches
      the transaction in `mIn`
    - `IPCThreadState::executeCommand` calls
      `IPCThreadState::doTransactBinder` to execute the transaction
    - `BBinder::transact` handles the transaction and calls
      virtual `BBinder::onTransact`
- java server
  - `Binder::transact` handles the transaction and calls virtual `Binder::onTransact`

## Old

binder (for IPC)
- every open to /dev/binder gives a binder_proc
- every thread has a stack (list actually).  One thread sends a transaction
  to the other by putting the transaction on its and remote's stack.  The remote is
  responsible for BC_FREE_BUFFER the transaction.
- basic types of binder transaction
  uint32_t[4]: 32bits unsigned integer
  string16[4+x]: an uint32_t specifies the length and followed by (len+1)*2 bytes
  flat object[4*4]: type, flag, pointer, cookie,
               where type can be BINDER, HANDLE, or FD
- binder can be strong or weak.  so can the corespondding handle.
- local filep can be sent to and installed by remote (with different fd)
- local data is a binder, remote data is a handle.
  local sends a binder to a remote and the binder becomes a handle to the remote
  remote sends a handle to another remote and the handle remains handle (with different id)
- context manager has a well-known handle: (void*)0.  everyone can send to context manager.
- e.g.
  WindowManager asks ServiceManager the handle of the SurfaceManager
  WindowManager sends transactions to SurfaceManager through the handle
- in kernel, every binder is a binder_node on local and a binder_ref on remote when sent
- when local write BC_TRANSACTION, the remote has a BINDER_WORK_TRANSACTION and
  the local has a BINDER_WORK_TRANSACTION_COMPLETE to read.
  If the transaction contains new binder(s), the local has BINDER_WORK_NODE.
- binder_node reference can be (local/internal, strong/weak)
- binder_ref reference can be strong/weak.
  BC_ACQUIRE/BC_INCREFS increases the strong/weak reference to binder_ref
  BC_RELEASE/BC_DECREFS decreases the strong/weak reference to binder_ref
  a strong refed binder_ref is a strong internal ref to binder_node
  a weak   refed binder_ref is a weak   internal ref to binder_node
- strong v.s. weak weak object can be used to compare the equality of two objects, or promote
  strong object can be invoked
  This is because weak object does not gurantee the lifetime of the object
- my guess on binder_node,
  binder_node has internal and local refs.
  internal ref is used by binder_ref
  local ref is used by kernel and userspace (object owner)
  transaction or work in kernel inc the local ref.
  userspace ref is not yet supported.  should that be, it uses BC_ATTEMPT_ACQUIRE?
  a binder_node is gone when there is no local ref and there is no remote (internal) ref
- my guess on binder_ref,
  binder_ref refs binder_node's internal ref
  note that binder_node does not have internal_weak_refs, instead, it has node->refs.
  binder_ref always weak ref binder_node by adding itself to node->refs (see binder_get_ref_for_node)
  binder_ref strong ref the node only once and only when binder_ref is strong refed.
- objects (BINDER or HANDLE) in transaction/work is locally refed.
  In binder_transaction, BINDER is binder_inc_node, HANDLE is binder_inc_ref.
- remote -> BC_TRANSACTION -> local -> BC_REPLY -> remote
  local should ACQUIRE needed objects and call BC_FREE_BUFFER to deref objects
  in transaction and free the buffer.  So does the remote after receiving REPLY.
- It is not possible for the local to locally ref an BINDER.  BINDER is
  on-demand.  maybe possible after ATTEMPT_INCREFS is implemented.

userspace and kernel
- a binder command consists of cmd and data
  if cmd is transaction, the data is (handle, code, flags, transaction data, offsets)
- binder object ptr/handle: RefBase::weakref_type in android
- binder object cookie: weakref's RefBase
- u: userspace, k: kernelspace, l: local, r: remote
  ur:BC_TRANSACTION (IPCThreadState::transact)
  kr:BINDER_WORK_TRANSACTION_COMPLETE
  kl:BINDER_WORK_TRANSACTION
  ur:BR_TRANSACTION_COMPLETE (IPCThreadState::waitForResponse)
  ul:BR_TRANSACTION (IPCThreadState::executeCommand)
  [local processes and returns]
  ul:BC_REPLY (IPCThreadState::sendReply)
  [kl:BINDER_WORK_NODE if local returns newly created object]
  kl:BINDER_WORK_TRANSACTION_COMPLETE
  kr:BINDER_WORK_TRANSACTION
  [ul:BR_{INCREFS,ACQUIRE,RELEASE,DECREFS} (IPCThreadState::executeCommand) if new objects]
  [ul:BC_{INCREFS,ACQUIRE}_DONE (IPCThreadState::executeCommand) if new objects]
  ul:BR_TRANSACTION_COMPLETE (IPCThreadState::waitForResponse)
  ur:BR_REPLY (IPCThreadState::waitForResponse)
- binder_inc_node takes a target_list to wait for userspace to ack the inc?
  local_strong_refs: is changed by kernel, and a work is scheduled
  has_strong_ref: is set in WORK_NODE together with pending
  pending_strong_ref: is set in WORK_NODE and sends BR_XXX to userspace
                      is cleared after userspace receives BR_XXX and sends BC_XXX.
- early after a new procees is spawned, a binder thread is created to join the
  pool.  This thread is BINDER_LOOPER_STATE_ENTERED.

mmap
- proc->buffer points to a continuous virtual kernel address
- proc->user_buffer_offset is negative number (vma start address - vmalloc start address)
- proc->pages: struct page **
- binder_update_page_range alloc_page() and map both vmalloc vm and userspace
  vm to the page.
- one zero-ed page is allocated and first binder_buffer is made allocated on
  that page.
- proc->buffers is a list of binder_buffer
- proc->free_buffers is sorted by buffer size.
- when transaction, a binder_transaction and a binder_buffer is
  binder_alloc_buf()ed on the TARGET proc.  One of the free_buffers is moved to
  allocated_buffers and if the free buffer is larger than needed size, it is
  splitted so that the rest is treated as another free buffer.

binder_proc
- has binder_thread(s), binder_node(s), binder_ref(s), binder_buffer(s)
- has binder_work(s) on todo, 
- has delivered_death

/proc/binder
- summary of phone app stats
- there are 11 binder_thread, 6 of them are either registered or entered
- there are 18 binder_node, all has_strong_ref and has_weak_ref
                            all nodes' local strong and local weak are 0
                            all nodes' internal strong and internal weak are 1, except one node
                            one node's internal strong and internal weak are 2 (by system_server and servicemanager)
- servicemanager holds refs for 4 of the 18 nodes, they are "phone", "iphonesubinfo", "simphonebook", and "isms" services
- there are 17 binder_ref.  all has strong and weak count 1.  one has non-zero death.
- there are more BR_TRANSACTION than BC_REPLY because some of the transactions are one-way.

binder_ref_death
- is a binder_work
- has a void __user *cookie, pointing to userspace struct.
- userspace is notified about a node's death when node owner close /dev/binder

binder_thread
- requested_threads: number of threads in request
  requested_threads_started: number of threads started
  That is, after a thread ioctl BC_REGISTER_LOOPER, requested_threads-- and requested_threads_started++.
  ready_threads: number of threads waiting to process proc->todo.
- when no ready thread and kernel haven't request any new thread,
  BR_SPAWN_LOOPER is sent.  userspace space should spawn a new thread and
  calls BC_REGISTER_LOOPER before looping.
- this is to guarantee proc->todo is handled immediately
- looper states
	BINDER_LOOPER_STATE_REGISTERED  = 0x01, /* thread called BC_REGISTER_LOOPER */
	BINDER_LOOPER_STATE_ENTERED     = 0x02, /* thread called BC_ENTER_LOOPER */
	BINDER_LOOPER_STATE_EXITED      = 0x04, /* */
	BINDER_LOOPER_STATE_INVALID     = 0x08, /* */
	BINDER_LOOPER_STATE_WAITING     = 0x10, /* */
	BINDER_LOOPER_STATE_NEED_RETURN = 0x20  /* thread should not wait for todo */

binder_transaction
- there are two parties: initiator and receiver
  there are two actions: send and reply
- initiator sends by pushing transaction to initiator's stack and links it to
  receiver's todo.  Unless oneway, where it is not pushed to initiator's stack.
  When any thread of the receiver is working on process todo, the transaction is
  pushed to the thread's stack if it is not oneway.
- receiver replys by popping initiating transaction from initiator's stack and
  links replying transaction to initiator's todo.  Before that, the initiating
  transaction on receiver's stack is popped as in_reply_to.
- to summarize, for non-oneway initiating binder_transaction, its "from"
  thread is always known.  Its "to_thread" is known after one of the
  receiver's thread takes care of it.  At that point, the transaction is
  on the top of initiator and receiver's stack.
- if A initiates a transaction X to B, and then B initiates a transaction Y to A,
  X.from == Y.to_thread and Y.from == X.to_thread.
- a thread MUST NOT send two transactions in a row.
  a transaction may contain another transaction.
  That is, parent/child transaction is allowed and siblings are forbidden.
- non-oneway transaction has t->from set to initiator's thread
- from:
    struct binder_proc *proc
    struct binder_thread *thread
    struct binder_transaction_data *tr
  target:
    struct binder_proc *target_proc;
    struct binder_thread *target_thread = NULL;
    struct binder_node *target_node = NULL;
    struct list_head *target_list;
    wait_queue_head_t *target_wait;
- target_thread is set if replying or 
- struct binder_transaction *t;


RefBase
- class RefBase::weakref_impl : public RefBase::weakref_type
  The class that has mStrong and mWeak as its members for ref counting
  {inc,dec}Weak takes an id only for debugging purpose (like who owns the ref).
- INITIAL_STRONG_VALUE: like floating reference in gobject
- object lifetime could be persistent or depending on weak or strong ref
- for each object, there is a weakref associated with it.  Their lifetimes may
  differ.  If object lifetime depends on strong ref, it is possible that weakref
  lives longer than object.
- weakref can be attemptIncStrong and it will succeed if the object is alive or
  has lifetime depend on weakref.  It is used to promote a wp.
- an object could be forceIncStrong regardless of its is dead or not.
- cmpxchg: if A is in charge of a resource, and we want to switch to B or C
           Both of B, C calls cmpxchg with value (A, B) and (A, C).
           Then either B or C will be in charge.
- attemptIncStrong: attemp to revive a object if it has a longer lifetime?
                    very similar to forceIncStrong?
- LightRefBase: simple ref counting

sp, wp
- basically, takes an object of type RefBase
- sp has m_ptr pointing to the object
- wp has m_ptr pointing to the object and m_refs pointing to object's weakref
  because object might die before weakref
- automagically {inc,dec}Strong and {inc,dec}Weak the object
- a wp can be promoted to a sp.  It might fail if the object is dead.

BBinder
- local object that can be used as a binder object (BINDER_TYPE_BINDER)
- class BBinder : public IBinder : public virtual RefBase
- remote IPCThreadState::transact -> local IPCThreadState::waitForResponse
  -> local IPCThreadState::executeCommand -> local BBinder::transact
- object attachment the same as BpBinder

BpBinder
- class BpBinder : public IBinder : public virtual RefBase
- ctor with a handle (object of type BINDER_TYPE_HANDLE)
  Has OBJECT_LIFETIME_WEAK and always has a weak ref to handle
- holds a strong ref to handle onFirstRef
- getInterfaceDescriptor: send INTERFACE_TRANSACTION
- pingBinder: send PING_TRANSACTION
- dump: send DUMP_TRANSACTION
- linkToDeath: takes a sp<DeathRecipient> which is stored as a wp.
               when reporting death, promote() before binderDied().
- attachObject: dict of (objectID, object) pairs.  NOT USED.

examples
- BnSurfaceComposer::CREATE_CONNECTION
  a sp<BClient> is created and inserted to parcel to become
  { .type = BINDER_TYPE_BINDER, .binder = local->getWeakRefs(), .cookie = local }
  where local is the BClient.  See flatten_binder.
  Since the BClient is newly created, local receives one of
  BR_INCREFS and BR_ACQUIRE.  Take BC_ACQUIRE for example,
  bclient is incStrong and BC_ACQUIRE_DONE is invoked
- in kernel ref count of BClient (strong l+i, weak l+i)
  (0+0, 0+0) when newly created in BC_REPLY
  (0+1, 0+0) when BC_REPLY because of new binder_ref and becomes a new work on reading todo
  (1+1, 1+0) when BINDER_WORK_NODE on reading todo and pending is set
  (0+1, 0+0) after userspace acks BR_{INCREFS,ACQUIRE} (does nothing) and when BC_{INCREFS,ACQUIRE}_DONE
  internal count: how many binder_ref there are.
  local count: how many transactions/works there are
- investigate
  BC_REPLY
  new node, new ref, and work is queued
  BR_INCREFS and BR_ACQUIRE
  BC_INCREFS_DONE and BC_INCREFS_DONE

interface
- class IInterface : public virtual RefBase
- class BpRefBase : public virtual RefBase
  ctor with a sp<IBinder>, and have both a strong and a weak ref on it
  The goal is to make those who own BpRefBase "own" the underlying IBinder.
  The underlying IBinder is gone when the last owner of BpRefBase is gone
- template<typename INTERFACE> class BnInterface : public INTERFACE, public BBinder
  e.g. SurfaceFlinter inherits BnSurfaceComposer, which inherits BnInterface<ISurfaceComposer>
- template<typename INTERFACE> class BpInterface : public INTERFACE, public BpRefBase
  e.g. remote creates BpSurfaceComposer, which inherits BpInterface<ISurfaceComposer>
- example
  class ISurfaceComposer : public IInterface
  class BpSurfaceComposer : public BpInterface<ISurfaceComposer>
  class BnSurfaceComposer : public BnInterface<ISurfaceComposer>
  Developer operates on sp<ISurfaceComposer>, which is a sp<IBinder> from
  servicemanager and interface_cast to sp<ISurfaceComposer>.
  How does interface_cast work?  It calls ISurfaceComposer::asInterface on the
  binder, which return a new BpSurfaceComposer, unless obj->queryLocalInterface
  returns something.  The latter happens only on BnInterface (local) binder.
- BpXXX is ctor with a sp<IBinder>, which is usually a BpBinder.
  e.g. BpSurfaceComposer(obj) -> BpInterface<ISurfaceComposer>(obj) ->
         BpRefBase(obj)
       BpSurfaceComposer is assigned to sp<ISurfaceComposer>.
  BpSurfaceComposer is OBJECT_LIFETIME_WEAK, and it holds both strong and weak ref to
  the underlying IBinder.  When assigned to sp<ISurfaceComposer>, sp incStrong.
  The seq is ISC owns BSC, which owns IB.  When ISC is gone,
  BSC->onLastStrongRef is called, which makes IB weak, and BSC is deleted, which makes
  IB gone.
  In another sense, ISC "owns" IB.

Parcel
- the payload of a transaction, like binder_io in servicemanager
- provides ways to read/write int, double, string, fd, and strong/weak binder
- writeObject  flattens a flat_binder_object to   an array of bytes
  readObject unflattens a flat_binder_object from an array of bytes
- writeStrongBinder calls   flatten_binder to   flatten a sp<IBinder>.
  readStrongBinder  calls unflatten_binder to unflatten a sp<IBinder>.
- writeWeakBinder and readWeakBinder promotes wp to sp and same as above.
- readXXXBinder needs to convert a handle to a BpBinder.
  It calls ProcessState::get{Strong,Weak}ProxyForHandle.
  
ProcessState
- opens and mmaps /dev/binder
- caches handle (BINDER_TYPE_HANDLE) to BpBinder mappings in mHandleToObject.
  as said, cache should not use strong ref.
- caches service name to BpBinder mappings in mContextObjects.
  Cache and hold strong ref as services are limited
  remember that handle 0 is service manager.
  Note that getContextObject sends invalid transaction to service manager.
  It means that it is _not_ used!
- see and use defaultServiceManager().
- expungeHandle?
- thread?

IPCThreadState
- on creation, pthread_setspecific(gTLS, this) itself.
- interface to /dev/binder
- e.g.
  freeBuffer -> sends BC_FREE_BUFFER
  flushCommands -> ioctl BINDER_WRITE_READ
  transact -> sends BC_TRANSACTION
  incStrongHandle -> sends BC_ACQUIRE
  decStrongHandle -> sends BC_RELEASE
  incWeakHandle -> sends BC_INCREFS
  decWeakHandle -> sends BC_DECREFS
  requestDeathNotification -> sends BC_REQUEST_DEATH_NOTIFICATION
- data are buffered in mIn and mOut first.
  They are sent after flush or waitForResponse
- thread?

binder class hierarchy
- wp, sp (weak/strong pointer or proxy?)
- class RefBase
  class BpRefBase : public virtual RefBase
  class IBinder : public virtual RefBase // Base class and low-level protocol for a remotable object.
  class BBinder : public IBinder
  class BpBinder : public IBinder
  class IInterface : public virtual RefBase
  class BnInterface : public INTERFACE, public BBinder
  class BpInterface : public INTERFACE, public BpRefBase
- defaultServiceManager(): invokes ProcessState::getContextObject, which sends
  a parcel and invokes readStrongBinder to get sp<IBinder>.
  interface_cast<IServiceManager> is called upon the ibinder to get a sp<IServiceManager>,
  where interface_cast does I##INTERFACE::asInterface: convert a IBinder to IINTERFACE: obj->queryLocalInterface or new BpINTERFACE
- For example, ServiceManager (remote makes calls to local):
  class IServiceManager : public IInterface // the interface
  class BpServiceManager : public BpInterface<IServiceManager> // the proxy used by remote
  class BnServiceManager : public BnInterface<IServiceManager> // running on local to listen to remote calls
  cmds/servicemanager/*.c // the impl., without using BnServiceManager
- to summary, to provide Olv, the formal way is:
  class Olv // the real impl.
  class IOlv : public IInterface // public interface of Olv
  class BpOlv : public BpInterface<IOlv> // the impl. used by remote, which makes calls to binder
  class BnOlv : public BnInterface<IOlv> // the demarshaller used by local, which makes calls to class Olv.

IBinder
- implemented by Binder and BinderProxy
- remote -> BinderProxy::transact -> BpBinder::transact -> write to /dev/binder
- 
