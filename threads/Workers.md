# Workers

`#include <cerver/threads/worker.h>`

A worker is a thread that takes the jobs of a [[job queue|Jobs]] and runs them, with states that can be stopped, resumed and ended from other threads. A worker has its own queue, or shares one with other workers, like the ones of a [[manager|Managers]].

## Concepts

### States

```
NONE ──worker_start ()──▶ AVAILABLE ◀──▶ WORKING
  │                         │   ▲
  └─worker_start_with_state (STOPPED)
                            ▼   │ worker_resume ()
                          STOPPED
AVAILABLE / WORKING / STOPPED ──worker_end ()──▶ ENDED
```

- A worker is `AVAILABLE` while it waits for a job, and `WORKING` while it runs one.
- `worker_stop ()` waits until the job that it runs ends, and then the worker takes no more jobs (`STOPPED`) until `worker_resume ()`.
- `worker_end ()` waits for the job that it runs and joins its thread (`ENDED`).
- Every change is made with the mutex of the worker and signaled in its `state_changed` condition.

### Jobs

A job of a worker runs the `work` of the worker with the `args` of the job (`worker_push_job ()`), or a work of its own (`worker_push_job_with_work ()`). `worker_set_delete_data ()` releases the arguments after the work. `worker_set_reference ()` keeps a pointer that the worker releases with `remove_reference` when it ends.

## Usage

``` c
Worker *worker = worker_create ();
worker_set_name (worker, "resizer");
worker_set_work (worker, resize_image);
worker_set_delete_data (worker, free);

(void) worker_start (worker);
(void) worker_push_job (worker, image_request);

(void) worker_end (worker);
worker_delete (worker);
```

## Reference

#### `worker_create ()`, `worker_create_with_id ()`, `worker_create_with_queue ()`, `worker_delete ()`

``` c
Worker *worker_create (void);
Worker *worker_create_with_id (const unsigned int id);
Worker *worker_create_with_queue (const unsigned int id, JobQueue *job_queue);
void worker_delete (void *worker_ptr);
```

Create a worker with its own queue, with an id, or taking its jobs from a queue that it shares and does not delete; end a worker if it runs and delete it.

#### `worker_set_name ()`, `worker_set_work ()`, `worker_set_delete_data ()`, `worker_set_reference ()`, `worker_set_remove_reference ()`

``` c
void worker_set_name (Worker *worker, const char *name);
void worker_set_work (Worker *worker, void (*work) (void *args));
void worker_set_delete_data (Worker *worker, void (*delete_data) (void *args));
void worker_set_reference (Worker *worker, const void *reference);
void worker_set_remove_reference (Worker *worker, void (*remove_reference) (const void *args));
```

Set the name of the worker (up to `WORKER_NAME_SIZE - 1` characters, the name of its thread), the work of its jobs, how their arguments are released, and a reference that it keeps and how it is released.

#### `worker_start ()`, `worker_start_with_state ()`

``` c
unsigned int worker_start (Worker *worker);
unsigned int worker_start_with_state (Worker *worker, const WorkerState worker_state);
```

Start the thread of the worker, `AVAILABLE`, or in `AVAILABLE` or `STOPPED`.

- **Return** 0 on success, 1 on error.

#### `worker_stop ()`, `worker_resume ()`, `worker_end ()`

``` c
unsigned int worker_stop (Worker *worker);
unsigned int worker_resume (Worker *worker);
unsigned int worker_end (Worker *worker);
```

Stop, resume and end a worker (see above).

- **Return** 0 on success, 1 if the worker was not in a state for it.

#### `worker_push_job ()`, `worker_push_job_with_work ()`

``` c
unsigned int worker_push_job (Worker *worker, void *args);
unsigned int worker_push_job_with_work (Worker *worker, void (*work) (void *args), void *args);
```

Add a job to the queue of the worker, with the work of the worker or with its own.

- **Return** 0 on success, 1 on error.

#### `worker_get_state ()`, `worker_state_to_string ()`, `worker_get_idle_milliseconds ()`

``` c
WorkerState worker_get_state (Worker *worker);
const char *worker_state_to_string (const WorkerState state);
u64 worker_get_idle_milliseconds (Worker *worker);
```

The state of a worker, its name, and the milliseconds that an available worker has been without a job (0 if it is not available).

## Types

| Type | What it is |
|---|---|
| `Worker` | `id`, `name`, `state`, its thread, its `job_queue` (and whether it owns it), its work, its reference, its mutex and condition |
| `WorkerState` | `WORKER_STATE_NONE`, `AVAILABLE`, `WORKING`, `STOPPED`, `ENDED` |

## Constants

| Constant | Value |
|---|---|
| `WORKER_NAME_SIZE` | 64 |

## See also

- [[Managers]]: workers that scale with their jobs
- [[Jobs]]: the queues of the workers
