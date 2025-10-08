# Threadpool

`#include <cerver/threads/thpool.h>`

A pool of threads with a **bounded** queue of jobs. The queue is a ring of a fixed size, allocated once, so a full pool rejects or waits instead of growing without a limit. A cerver with `CERVER_HANDLER_TYPE_THREADS` uses one for its connections (see [[Cerver]]); an application can create its own.

## Concepts

### Adding work

| Function | When the queue is full |
|---|---|
| `thpool_add_work ()` | Returns 1 right away |
| `thpool_add_work_with_reject ()` | Returns 1 right away; the job has a `reject` for when it is dropped later |
| `thpool_add_work_wait ()` | Waits until there is space |

A job that was queued but is dropped before it runs, by a drain that rejects or by the destroy, gets its `reject` with its `args` instead of its `work`, so its arguments can always be released.

### Draining

`thpool_drain ()` stops accepting jobs and waits for the ones in progress. With `run_queued`, the queued jobs still run until the timeout; without it, they are rejected right away. After the timeout, the queued jobs that are left are rejected. `thpool_destroy ()` stops the threads after their current job and rejects what is still queued.

### Stats

A pool counts its jobs (total, completed, rejected, dropped, the times that a caller waited, the most that were queued), the total time that they waited and ran, and two histograms of those times in buckets of up to 2^0, 2^1 … 2^18 milliseconds, with a last bucket for the longer ones.

## Usage

``` c
static void my_work (void *args) {

	(void) args;

}

static void my_reject (void *args) {

	free (args);

}

Thpool *pool = thpool_create (8);
thpool_set_name (pool, "my-pool");
(void) thpool_set_queue_size (pool, 64);

if (!thpool_init (pool)) {
	if (thpool_add_work_with_reject (pool, my_work, my_reject, malloc (64))) {
		// the queue is full
	}

	(void) thpool_drain (pool, true, 8);

	ThpoolStats stats;
	thpool_get_stats (pool, &stats);
	thpool_stats_print (&stats);
}

thpool_destroy (pool);
```

## Reference

### Create and run

#### `thpool_create ()`

``` c
Thpool *thpool_create (unsigned int n_threads);
```

Creates a pool of `n_threads` threads, not started yet, with a queue of `n_threads * THPOOL_DEFAULT_QUEUE_SIZE_FACTOR` jobs, rounded up to a power of 2.

- **Returns** the pool, or `NULL` on error.

#### `thpool_set_queue_size ()`

``` c
unsigned int thpool_set_queue_size (Thpool *thpool, unsigned int queue_size);
```

Sets how many jobs can wait in the queue, rounded up to a power of 2, before `thpool_init ()`.

- **Returns** 0 on success, 1 on error.

#### `thpool_set_name ()`

``` c
void thpool_set_name (Thpool *thpool, const char *name);
```

Sets the name of the pool, up to `THPOOL_NAME_SIZE - 1` characters, used to name its threads.

#### `thpool_init ()`

``` c
unsigned int thpool_init (Thpool *thpool);
```

Starts the threads, and waits until every one is running.

- **Returns** 0 on success, 1 on error, with no thread left running.

#### `thpool_destroy ()`

``` c
void thpool_destroy (Thpool *thpool);
```

Stops the threads after their current job, rejects every job that is still queued, and deletes the pool.

### Work

#### `thpool_add_work ()`, `thpool_add_work_with_reject ()`

``` c
int thpool_add_work (Thpool *thpool, void (*work) (void *), void *args);
int thpool_add_work_with_reject (Thpool *thpool, void (*work) (void *), void (*reject) (void *), void *args);
```

Queue a job without blocking.

- **Return** 0 on success, 1 if the queue is full or the pool is not accepting jobs; then the caller still owns `args`.

#### `thpool_add_work_wait ()`

``` c
int thpool_add_work_wait (Thpool *thpool, void (*work) (void *), void (*reject) (void *), void *args);
```

Queues a job, waiting while the queue is full.

- **Returns** 0 on success, 1 if the pool stopped accepting jobs.

#### `thpool_wait_space ()`

``` c
int thpool_wait_space (Thpool *thpool, int timeout);
```

Waits until a job can be queued, the pool stops accepting jobs, `timeout` milliseconds pass, or a signal interrupts the wait; it waits with `poll ()` on an eventfd, so a signal can interrupt it.

- **Returns** 0 when there is space, 1 on timeout or when the pool is not accepting jobs, and -1 with `errno` `EINTR` when a signal interrupted it.

#### `thpool_wait ()`

``` c
void thpool_wait (Thpool *thpool);
```

Waits until every job has finished.

#### `thpool_drain ()`

``` c
unsigned int thpool_drain (Thpool *thpool, bool run_queued, time_t timeout);
```

Stops accepting jobs and waits for the current ones (see above). A timeout of `THPOOL_DRAIN_TIMEOUT_NONE` (0) waits until every job has finished.

- **Returns** 0 if every job finished, 1 if some threads are still working.

### State and stats

#### `thpool_get_num_threads_alive ()`, `thpool_get_num_threads_working ()`, `thpool_get_queued ()`, `thpool_get_queue_size ()`

``` c
unsigned int thpool_get_num_threads_alive (Thpool *thpool);
unsigned int thpool_get_num_threads_working (Thpool *thpool);
unsigned int thpool_get_queued (Thpool *thpool);
unsigned int thpool_get_queue_size (Thpool *thpool);
```

The threads that are running, the ones that are working in a job, the jobs that wait in the queue, and the size of the queue.

#### `thpool_is_empty ()`, `thpool_is_full ()`

``` c
bool thpool_is_empty (Thpool *thpool);
bool thpool_is_full (Thpool *thpool);
```

`true` without working threads, and `true` when a new job can not be queued right now (the queue is full or the pool is not accepting jobs).

#### `thpool_get_stats ()`, `thpool_stats_print ()`

``` c
void thpool_get_stats (Thpool *thpool, ThpoolStats *stats);
void thpool_stats_print (const ThpoolStats *stats);
```

Copy the stats of the pool, and print them with the log.

#### `thpool_histogram_bucket ()`

``` c
unsigned int thpool_histogram_bucket (const u64 microseconds);
```

Returns the bucket of the histograms for a time in microseconds.

## Types

| Type | What it is |
|---|---|
| `Thpool` | A pool: its name, its threads, its queue and its stats |
| `ThpoolJob` | `work`, `reject`, `args` and when it was queued |
| `ThpoolStats` | `total_jobs`, `completed_jobs`, `rejected_jobs`, `dropped_jobs`, `waited_jobs`, `max_queued`, the total wait and run times, and their histograms |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `THPOOL_DEFAULT_QUEUE_SIZE_FACTOR` | 2 | The default queue is the threads times it |
| `THPOOL_DRAIN_TIMEOUT_NONE` | 0 | A drain without a timeout |
| `THPOOL_HISTOGRAM_MAX_EXPONENT` | 18 | The last bucket of up to 2^18 ms |
| `THPOOL_HISTOGRAM_BUCKETS` | 20 | The buckets of the histograms |
| `THPOOL_NAME_SIZE` | 64 | The buffer of the name |

## See also

- [[Cerver]]: the thpool of a cerver and its overload policy
- [[Jobs]]: queues of jobs with priorities
- [[Managers]]: workers that scale with their jobs
