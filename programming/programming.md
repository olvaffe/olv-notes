Programming
===========

## A new project

- must have when applicable
  - c99:
    - <http://www.open-std.org/jtc1/sc22/wg14/www/newinc9x.htm>
    - `inline`
    - designated initializer
    - stdbool and stdint
    - trailing comma in enum
    - `__VA_ARGS__`
    - `__function__`
  - `log.c` for logging
    - log destinations (stderr, syslog, ...)
    - log levels (as used by syslog)
  - `utils.c` for data structures
    - kernel list
    - kernel kref
    - hash table
  - `loop.c` for event loop
    - IO
    - Timer
    - Signal
    - Idle
    - Watch out for source removal during dispatching.  It may happen that both
      source N and N+k are to be dispatched, but source N removes source N+k
      before the dispatch of source N+k happens.
    - Also note that a fd should be EPOLL_CTL_DEL before closing
    - Another thing to consider is that, how the program cleans up itself after
      quitting the loop.
    - Do not necessarily need to use the main loop for all signals.
  - `settings.c` for project wide configuration
    - `const struct proj_settings *proj_get_settings(void);`
- high-level structures and low-level structures
  - high-level structures has pointers to low-level structures
  - low-level structures _must_ not know about the high-level strucutres
    - otherwise, they won't be low-level anymore
  - reference count the low-level strucutres, not the high-level ones
    - this allows the low-level structures to be passed around
- Reference counting
  - the number of current users of an object
    - garbage collect an object when the count reaches zero
    - on the other hand, the object always exists as long as there are users
  - a must if an object is used by multiple threads concurrently
    - otherwise, imagine how to handle one thread calling obj_do_someting and
      the other thread calling obj_destroy
  - how is it useful for non-threaded environment?
    - Garbage collection.  If object A is created only for object B, we may want
      object A to be destroyed automatically with B.
    - Async operation.  Suppose destroying object A requires killing a child
      process, which takes time.  We may want the object to exist a little long
      until the child process is terminated.
  - If object A automatically creates many object B's, and each object B owns a
    reference of object A, how do we destroy object A without manually
    destroying B's first?
    - We should avoid this.  Instead, require B to have a shorter lifetime than
      A.  B can be automatically destroyed with A.
    - If B is reference counted, it should not point back to A.
- Error handling
  - how to propogate errors to the caller
  - `GError`?

## signness

signed pros:

int size = sizeof(entry) * n_entries;
if (size <= 0) { /* this check is not abnormal */
	return;
}

unsigned size = sizeof(entry) * n_entries;
if (size <= n_entries) { /* this check is abnormal */
	return;
}

## old

Every source file should know

1. what does it interface
2. what interface does it have

That means, I should know what I expect to use this file _and_ what is the
underlying component (hw, device node, library, etc.) capable of.  The
interface it has should be easy to use, but not over worked, which is a sign of
bad modeling.  There is nothing wrong if the underlying component is part of
the interface.  It depends on the need.  When this is resolved (it is an
incremental process), I could move on to implementation stage.

Having a good understanding of 1 and 2, I should know what functions from the
underlying component are needed and what are not.  See, this is the process of
transforming one thing to another which is more suitable for the need.  It is
not uncommon only a small subset of the functions is needed.

A good implementation should

3. have a layer which guarantees direct and faithful calls to needed functions
   of the underlying component.

Usually, not every function is directly usable.  None is directly usable if we
are writing a driver.  But one should note that no smart things are _allowed_
in this layer.  Faithful is important.  Above this layer, we could implement
the interface we want to provide.  Note that it is perfectly legal part of this
layer is part of the interface.

For whole project, two things are important

4. Logging mechanism
5. Statistics

An example.  Say we have an application, app.  We are writing aaa.c,
interfacing component bbb (could be another source file, a hardware, or a
library).  The interface is like:

#ifndef _AAA_H_
#define _AAA_H_

#if DEPEND_ON_WHAT_I_NEED
#include <bbb.h>
#endif

void app_aaa_do_something(void);
void app_aaa_another_thing(void);

/* these are allowed */
extern int app_aaa_global_variable;
void _app_aaa_ugly_hack(void); /* unless this is a public header.. */

/* both could be part of the thin layer in 3! */
void app_aaa_yet_another(void);
void app_aaa_do_bbb(void);

#endif /* _AAA_H_ */

Underscore is for functions that are not well-considered.  Global functions
could have underscore (and they are called hacks).  Static functions might or
might not have underscore.  It is about the maturity of the function, not the
scope.  But one should be more careful when writing a library.  Underscore'ed
global functions should at least not in the public headers.

aaa.c:

#include "aaa.h"

int app_aaa_global_variable;

void app_aaa_do_something(void)
{
}

static void app_aaa_helper(void)
{
}

void app_aaa_another_thing(void)
{
}

void _app_aaa_ugly_hack(void)
{
}

/* these are in thin layer */
void app_aaa_yet_another(void)
{
}

void app_aaa_do_bbb(void)
{
}

/* more thin layer stuff; static this time */
static void bbb_yoyo(void)
{
}

static void bbb_yeye(void)
{
}

static void _bbb_whatever(void)
{
}
