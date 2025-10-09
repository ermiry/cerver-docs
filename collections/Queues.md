# Queues

`#include <cerver/collections/queue.h>`

A `Queue` is a first in, first out queue of pointers on top of a [[list|Lists]], with a destroy function for its data. For a queue of jobs with priorities and waits, see [[Jobs]].

## Usage

``` c
Queue *queue = queue_create (free);

(void) queue_push (queue, strdup ("first"));
(void) queue_push (queue, strdup ("second"));

char *first = (char *) queue_pop (queue);
free (first);

queue_delete (queue);
```

## Reference

``` c
Queue *queue_create (void (*destroy)(void *data));
int queue_init (Queue *queue, void *(*create)(void), unsigned int n_elements);
void queue_delete (Queue *queue);
```

Create a queue, fill it with `n_elements` elements of `create` (0 on success, 1 if one failed), and delete it with its data.

``` c
unsigned int queue_push (Queue *queue, void *data);
void *queue_pop (Queue *queue);
size_t queue_size (Queue *queue);
```

Add data at the end (0 on success, 1 on error), take the oldest data (`NULL` when it is empty), and the elements in the queue.

``` c
void queue_reset (Queue *queue);
void queue_clear (Queue *queue);
```

Release the elements with their data, or only the elements, keeping the queue.

## Types

| Type | What it is |
|---|---|
| `Queue` | `dlist`, `destroy` |

## See also

- [[Lists]]: the list of a queue
- [[Jobs]]: queues of jobs for threads
