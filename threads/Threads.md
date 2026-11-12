# Threads

`#include <cerver/threads/thread.h>`

The threads that cerver creates, and the helpers that it uses to create threads, name them and allocate mutexes and conditions. The pools of threads are in [[Threadpool]], the queues of jobs in [[Jobs]], and the workers in [[Workers]] and [[Managers]].

## Concepts

### Signals

**Every thread that cerver creates blocks `SIGINT` and `SIGTERM`**, so the signals of the process always reach the main thread, where the application handles them (like with `cerver_shutdown ()`, see [[Lifecycle]]). A thread that the application creates with these functions does the same.

### Thread groups

A `ThreadGroup` counts the detached threads that use the data of a cerver or a client, like the events that run in their own threads: the teardown waits for them before it releases the data. The groups are internal; they are why an action of an event can always use its cerver while it runs.

### Names

`thread_set_name ()` names the calling thread, as `ps -L` and `gdb` show it; cerver names its own threads after the alias of the cerver, the name of the client and the id of the handler. Linux keeps the first 15 characters.

## Reference

#### `thread_create_joinable ()`

``` c
unsigned int thread_create_joinable (pthread_t *thread, void *(*work) (void *), void *args);
```

Creates a thread, that must be joined with `pthread_join ()`.

- **Returns** 0 on success, 1 on error.

#### `thread_create_detachable ()`, `thread_create_detached ()`

``` c
unsigned int thread_create_detachable (pthread_t *thread, void *(*work) (void *), void *args);
pthread_t thread_create_detached (void *(*work) (void *), void *args);
```

Create a detached thread, that goes away on its own when its work returns; the first returns 0 on success and 1 on error, the second returns the thread, or 0 on error.

#### `thread_set_name ()`

``` c
unsigned int thread_set_name (const char *name, ...);
```

Names the calling thread with a format and its arguments, like `printf ()`.

- **Returns** 0 on success, 1 on error.

#### `thread_mutex_new ()`, `thread_mutex_delete ()`, `thread_mutex_lock ()`, `thread_mutex_unlock ()`

``` c
pthread_mutex_t *thread_mutex_new (void);
void thread_mutex_delete (pthread_mutex_t *mutex);
void thread_mutex_lock (pthread_mutex_t *mutex);
void thread_mutex_unlock (pthread_mutex_t *mutex);
```

Allocate and initialize a mutex, destroy and release it, and lock and unlock it.

#### `thread_cond_new ()`, `thread_cond_delete ()`

``` c
pthread_cond_t *thread_cond_new (void);
void thread_cond_delete (pthread_cond_t *cond);
```

Allocate and initialize a condition, and destroy and release it.

## Types

| Type | What it is |
|---|---|
| `ThreadGroup` | The threads that are `alive`, with its `mutex` and its `ended` condition |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `THREAD_OK` | 0 | |
| `THREAD_NAME_BUFFER_SIZE` | 64 | The buffer of a name before it is cut |
| `THREAD_GROUP_WAIT_FOREVER` | 0 | A wait for a group without a timeout |

## See also

- [[Threadpool]], [[Jobs]], [[Workers]], [[Managers]], [[Semaphores]]
