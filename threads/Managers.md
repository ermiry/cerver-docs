# Managers

`#include <cerver/threads/manager.h>`

A `WorkerManager` is a group of [[Workers]] that take their jobs from the same [[queue|Jobs]], with a control thread that **scales** them: it adds a worker while the jobs wait without a free one, and removes the workers that are idle, always between a minimum and a maximum.

## Concepts

- **Scaling up:** a job that waited `scale_up_wait` milliseconds (64 by default) without a free worker makes the manager add one, up to the maximum.
- **Scaling down:** a worker without a job for `idle_timeout` milliseconds (4096 by default) is removed, one per check, down to the minimum. No worker is removed while jobs are waiting in the queue.
- **Checks:** the control thread checks the workers every `WORKER_MANAGER_DEFAULT_CHECK_INTERVAL` (128) milliseconds, and when a job is pushed.
- **Stop and resume:** `worker_manager_stop ()` stops every worker after its job, and the jobs stay in the queue until `worker_manager_resume ()`.

## Usage

``` c
WorkerManager *manager = worker_manager_create ("images");

(void) worker_manager_set_limits (manager, 2, 16);
worker_manager_set_scaling (manager, 32, 8192);
worker_manager_set_work (manager, resize_image);

if (!worker_manager_start (manager)) {
	(void) worker_manager_push_job (manager, NULL, image_request);
	(void) worker_manager_push_job_priority (manager, NULL, urgent_request, JOB_QUEUE_PRIORITY_HIGH);
}

(void) worker_manager_end (manager);
worker_manager_delete (manager);
```

## Reference

#### `worker_manager_create ()`, `worker_manager_delete ()`

``` c
WorkerManager *worker_manager_create (const char *name);
void worker_manager_delete (WorkerManager *manager);
```

Create a manager with a queue of `JOB_QUEUE_DEFAULT_PRIORITIES` priorities, and end it if it runs and delete it with its workers and its queue.

#### `worker_manager_set_limits ()`

``` c
unsigned int worker_manager_set_limits (WorkerManager *manager, const unsigned int min_workers, const unsigned int max_workers);
```

Sets the least and the most workers, from 1 to `WORKER_MANAGER_MAX_WORKERS` (256), before the start. The defaults are 1 and 8.

- **Returns** 0 on success, 1 if the limits are not valid.

#### `worker_manager_set_scaling ()`

``` c
void worker_manager_set_scaling (WorkerManager *manager, const u32 scale_up_wait, const u32 idle_timeout);
```

Sets when a worker is added and when one is removed, in milliseconds (see above).

#### `worker_manager_set_work ()`

``` c
void worker_manager_set_work (WorkerManager *manager, void (*work) (void *args));
```

Sets the work of the jobs that are pushed without one.

#### `worker_manager_start ()`, `worker_manager_stop ()`, `worker_manager_resume ()`, `worker_manager_end ()`

``` c
unsigned int worker_manager_start (WorkerManager *manager);
unsigned int worker_manager_stop (WorkerManager *manager);
unsigned int worker_manager_resume (WorkerManager *manager);
unsigned int worker_manager_end (WorkerManager *manager);
```

Start the minimum workers and the control thread; stop the workers after their jobs; resume them; end the control thread and every worker after its job.

- **Return** 0 on success, 1 if the manager was not in a state for it.

#### `worker_manager_push_job ()`, `worker_manager_push_job_priority ()`

``` c
unsigned int worker_manager_push_job (WorkerManager *manager, void (*work) (void *args), void *args);
unsigned int worker_manager_push_job_priority (WorkerManager *manager, void (*work) (void *args), void *args, const unsigned int priority);
```

Push a job, with the default priority or with a priority (0 is the highest); `work` `NULL` uses the work of the manager.

- **Return** 0 on success, 1 on error.

#### `worker_manager_get_n_workers ()`, `worker_manager_get_states ()`, `worker_manager_get_stats ()`

``` c
unsigned int worker_manager_get_n_workers (WorkerManager *manager);
unsigned int worker_manager_get_states (WorkerManager *manager, WorkerState *states, const unsigned int max_states);
void worker_manager_get_stats (WorkerManager *manager, WorkerManagerStats *stats);
```

The workers that the manager has now, the state of each one (up to `max_states`, returning how many were copied), and its stats.

## Types

| Type | What it is |
|---|---|
| `WorkerManager` | A manager: its name, queue, workers, limits, scaling and control thread |
| `WorkerManagerStats` | `workers`, `available`, `working`, `stopped`, `created_workers`, `removed_workers`, `max_workers` and the stats of its queue |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `WORKER_MANAGER_MAX_WORKERS` | 256 | |
| `WORKER_MANAGER_DEFAULT_MIN_WORKERS`, `WORKER_MANAGER_DEFAULT_MAX_WORKERS` | 1, 8 | |
| `WORKER_MANAGER_DEFAULT_SCALE_UP_WAIT` | 64 | Milliseconds that a job waits before a worker is added |
| `WORKER_MANAGER_DEFAULT_IDLE_TIMEOUT` | 4096 | Milliseconds idle before a worker is removed |
| `WORKER_MANAGER_DEFAULT_CHECK_INTERVAL` | 128 | Milliseconds between the checks |
| `WORKER_MANAGER_NAME_SIZE` | 64 | |

## See also

- [[Workers]]: the workers of a manager
- [[Threadpool]]: a fixed pool of threads with a bounded queue
