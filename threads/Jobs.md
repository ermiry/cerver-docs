# Jobs

`#include <cerver/threads/jobs.h>`

A `JobQueue` is a queue of jobs with **priorities**, a condition to wait for them, and a pool of jobs to reuse them. The handlers with a queue use one (see [[Handlers]]), and so do the send queues of the connections, the [[Workers]] and the [[Managers]].

## Concepts

### Priorities

A queue has from 1 to `JOB_QUEUE_MAX_PRIORITIES` (16) priorities, 3 by default; 0 is the highest. `job_queue_push ()` uses the one in the middle (`job_queue_default_priority ()`), and the pulls take the highest priority first. So that a low priority is never starved, after `JOB_QUEUE_DEFAULT_STARVATION_LIMIT` (8) jobs were taken ahead of a waiting job of a lower priority, the next one is taken from the lower priority. Each priority is a ring that grows by doubling.

### Waiting

| Function | Waits |
|---|---|
| `job_queue_pull ()` | Never: `NULL` if the queue is empty |
| `job_queue_pull_timeout ()` | Up to a timeout in milliseconds, or `JOB_QUEUE_WAIT_FOREVER` |
| `job_queue_pull_interruptible ()` | Until a job arrives or a predicate says to stop |
| `job_queue_request ()`, `job_queue_request_timeout ()` | For the job with an id, pushed with `job_queue_push_job_with_id ()` |

`job_queue_signal ()` wakes up every thread that waits: they return `NULL`. A signal that arrives before a thread starts to wait is not lost. `job_queue_pull_interruptible ()` calls its predicate with the lock of the queue, so the thread that changes what it reads has to call `job_queue_signal ()` after the change.

### Jobs from the pool

`job_get ()` takes a job from the pool of the queue and `job_return ()` gives it back, so pushing a job does not allocate. `job_create ()` allocates one.

### Stats

A queue counts its jobs (total, pulled, completed, requested, the waits that timed out, the jobs that were promoted against starvation, the most queued, the jobs of each priority), and the total and the histograms of the times that they waited and ran, like a [[Threadpool]]. The run time is added with `job_queue_record_run ()`.

## Usage

``` c
JobQueue *queue = job_queue_create (JOB_QUEUE_TYPE_JOBS);

(void) job_queue_push_job_priority (queue, my_work, my_args, JOB_QUEUE_PRIORITY_HIGH);

Job *job = (Job *) job_queue_pull_timeout (queue, 1000);
if (job) {
	struct timespec start;
	(void) clock_gettime (CLOCK_MONOTONIC, &start);

	job->work (job->args);

	job_queue_record_run (queue, &start);
	job_return (queue, job);
}

job_queue_delete (queue);
```

## Reference

### Queues

#### `job_queue_create ()`, `job_queue_create_with_priorities ()`, `job_queue_new ()`, `job_queue_delete ()`

``` c
JobQueue *job_queue_create (const JobQueueType type);
JobQueue *job_queue_create_with_priorities (const JobQueueType type, const unsigned int n_priorities);
JobQueue *job_queue_new (void);
void job_queue_delete (void *job_queue_ptr);
```

Create a queue of `Job`s (`JOB_QUEUE_TYPE_JOBS`) or of `JobHandler`s (`JOB_QUEUE_TYPE_HANDLERS`), with the default priorities or with `n_priorities`; allocate an empty one; delete a queue with the jobs that it still has.

#### `job_queue_set_starvation_limit ()`, `job_queue_default_priority ()`

``` c
void job_queue_set_starvation_limit (JobQueue *job_queue, const unsigned int limit);
unsigned int job_queue_default_priority (const JobQueue *job_queue);
```

Set how many jobs can be taken ahead of a waiting job of a lower priority (`JOB_QUEUE_STARVATION_DISABLED`, 0, always takes the highest priority first), and return the priority in the middle.

#### `job_queue_size ()`, `job_queue_oldest_wait ()`

``` c
size_t job_queue_size (JobQueue *job_queue);
u64 job_queue_oldest_wait (JobQueue *job_queue);
```

The jobs in the queue, and the milliseconds that the oldest one has waited (0 when it is empty).

#### `job_queue_push ()`, `job_queue_push_priority ()`

``` c
unsigned int job_queue_push (JobQueue *job_queue, void *job_ptr);
unsigned int job_queue_push_priority (JobQueue *job_queue, void *job_ptr, const unsigned int priority);
```

Add a job with the default priority, or with a priority; one that does not exist is the lowest.

- **Return** 0 on success, 1 on error; then the caller still owns the job.

#### `job_queue_push_job ()`, `job_queue_push_job_priority ()`, `job_queue_push_job_with_id ()`

``` c
unsigned int job_queue_push_job (JobQueue *job_queue, void (*work) (void *args), void *args);
unsigned int job_queue_push_job_priority (JobQueue *job_queue, void (*work) (void *args), void *args, const unsigned int priority);
unsigned int job_queue_push_job_with_id (JobQueue *job_queue, const u64 job_id, void (*work) (void *args), void *args);
```

Create a job and add it; with an id, a thread that waits for it in `job_queue_request ()` gets it directly.

- **Return** 0 on success, 1 on error.

#### `job_queue_push_handler ()`

``` c
unsigned int job_queue_push_handler (
	JobQueue *job_queue,
	Cerver *cerver, Connection *connection,
	void *data, void (*data_delete) (void *data_ptr)
);
```

Adds a `JobHandler` to a queue of handlers.

- **Returns** 0 on success, 1 on error.

#### `job_queue_pull ()`, `job_queue_pull_timeout ()`, `job_queue_pull_interruptible ()`

``` c
void *job_queue_pull (JobQueue *job_queue);
void *job_queue_pull_timeout (JobQueue *job_queue, const u32 timeout);
void *job_queue_pull_interruptible (JobQueue *job_queue, bool (*interrupted) (const void *args), const void *args);
```

Take the next job (see the waits above).

- **Return** the job, or `NULL`.

#### `job_queue_request ()`, `job_queue_request_timeout ()`

``` c
void *job_queue_request (JobQueue *job_queue, const u64 job_id);
void *job_queue_request_timeout (JobQueue *job_queue, const u64 job_id, const u32 timeout);
```

Wait for the job with an id, without a limit or up to `timeout` milliseconds.

- **Return** the job, or `NULL` if it did not arrive.

#### `job_queue_wait ()`, `job_queue_signal ()`

``` c
void job_queue_wait (JobQueue *job_queue);
void job_queue_signal (JobQueue *job_queue);
```

Wait until the queue has a job or is signaled, and wake up every thread that waits.

#### `job_queue_set_handler ()`, `job_queue_start ()`, `job_queue_stop ()`

``` c
void job_queue_set_handler (JobQueue *queue, void (*handler) (void *data));
unsigned int job_queue_start (JobQueue *job_queue);
unsigned int job_queue_stop (JobQueue *job_queue);
```

Give a queue a handler and a thread that runs it with each job, start it and stop it (0 on success, 1 on error).

#### `job_queue_clear ()`

``` c
void job_queue_clear (JobQueue *job_queue);
```

Deletes every job of the queue.

#### `job_queue_record_run ()`, `job_queue_get_stats ()`, `job_queue_stats_print ()`

``` c
void job_queue_record_run (JobQueue *job_queue, const struct timespec *start);
void job_queue_get_stats (JobQueue *job_queue, JobQueueStats *stats);
void job_queue_stats_print (JobQueue *job_queue);
```

Add the run of a job that started at `start` (`CLOCK_MONOTONIC`), copy the stats, and print them.

### Jobs

#### `job_create ()`, `job_new ()`, `job_delete ()`, `job_reset ()`, `job_comparator ()`

``` c
Job *job_create (void (*work) (void *args), void *args);
void *job_new (void);
void job_delete (void *job_ptr);
void job_reset (Job *job);
int job_comparator (const void *a, const void *b);
```

Allocate a job with its work, allocate an empty one, delete one, clear one, and compare two by their ids.

#### `job_get ()`, `job_return ()`

``` c
Job *job_get (JobQueue *job_queue);
void job_return (JobQueue *job_queue, Job *job);
```

Take a job from the pool of a queue (it allocates one only if the pool is empty), and give it back.

### Job handlers

#### `job_handler_new ()`, `job_handler_create ()`, `job_handler_delete ()`, `job_handler_get ()`, `job_handler_reset ()`, `job_handler_return ()`

``` c
void *job_handler_new (void);
void *job_handler_create (void);
void job_handler_delete (void *job_handler_ptr);
JobHandler *job_handler_get (JobQueue *job_queue);
void job_handler_reset (JobHandler *job_handler);
void job_handler_return (JobQueue *job_queue, JobHandler *job_handler);
```

The same for the `JobHandler`s: a job with a cerver, a connection and data, and a condition that its creator waits on.

#### `job_handler_wait ()`, `job_handler_signal ()`

``` c
void job_handler_wait (JobQueue *job_queue, void *data, void (*data_delete) (void *data_ptr));
void job_handler_signal (JobHandler *handler);
```

Push a handler with the data and wait until it is done, and mark a handler as done and wake up its creator.

## Types

| Type | What it is |
|---|---|
| `JobQueue` | A queue: its type, its pool, its rings by priority, its condition, its stats |
| `JobQueueType` | `JOB_QUEUE_TYPE_NONE`, `JOBS`, `HANDLERS` |
| `Job` | `id`, `work`, `args` |
| `JobHandler` | `cerver`, `connection`, `cond`, `mutex`, `done`, `data`, `data_delete` |
| `JobQueueStats` | The counts and the times of the jobs, see above |
| `JobQueueRing`, `JobQueueEntry` | The ring of a priority and its entries |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `JOB_QUEUE_DEFAULT_PRIORITIES` | 3 | |
| `JOB_QUEUE_MAX_PRIORITIES` | 16 | |
| `JOB_QUEUE_PRIORITY_HIGH`, `_NORMAL`, `_LOW` | 0, 1, 2 | The priorities of a default queue |
| `JOB_QUEUE_DEFAULT_RING_CAPACITY` | 16 | The first room of a priority |
| `JOB_QUEUE_DEFAULT_STARVATION_LIMIT` | 8 | |
| `JOB_QUEUE_STARVATION_DISABLED` | 0 | |
| `JOB_QUEUE_WAIT_FOREVER` | `0xFFFFFFFF` | A wait without a deadline |
| `JOB_QUEUE_POOL_INIT` | 16 | The jobs of the pool of a queue |

## See also

- [[Handlers]]: the queues of the handlers
- [[Workers]] and [[Managers]]: threads that take jobs from a queue
