# Pools

`#include <cerver/collections/pool.h>`

A `Pool` keeps elements to be reused instead of allocated again: a stack on an array, where the last element that was returned is the first one that is taken, so it is the one that is still in the cache. Pushing and popping do not allocate; only growing past its capacity does. The job queues keep their jobs in pools.

## Concepts

- **Producing:** with `pool_set_produce_if_empty (pool, true)`, a pop from an empty pool creates an element with the `create` of the pool.
- **Limit:** `pool_set_max_elements ()` caps the elements that the pool keeps; an element returned to a full pool is destroyed. `POOL_NO_LIMIT` (the default) keeps all of them.
- The functions lock the mutex of the pool.

## Usage

``` c
Pool *buffers = pool_create (free);
pool_set_create (buffers, my_buffer_create);
pool_set_produce_if_empty (buffers, true);
pool_set_max_elements (buffers, 64);

void *buffer = pool_pop (buffers);
// use the buffer
(void) pool_push (buffers, buffer);

pool_delete (buffers);
```

## Reference

``` c
Pool *pool_create (void (*destroy)(void *data));
int pool_init (Pool *pool, void *(*create)(void), unsigned int n_elements);
void pool_delete (Pool *pool);
```

Create a pool, fill it with `n_elements` elements of `create` (`NULL` uses the one of the pool; 0 on success, 1 if one failed), and delete it with its elements.

``` c
void pool_set_destroy (Pool *pool, void (*destroy)(void *data));
void pool_set_create (Pool *pool, void *(*create)(void));
void pool_set_produce_if_empty (Pool *pool, bool produce);
void pool_set_max_elements (Pool *pool, size_t max_elements);
```

Set how the elements are destroyed and created, whether an empty pool creates one, and the most elements that it keeps.

``` c
int pool_push (Pool *pool, void *data);
void *pool_pop (Pool *pool);
size_t pool_size (Pool *pool);
```

Return an element to the pool (0 on success, 1 on error; an element that does not fit is destroyed), take the last one that was returned (`NULL` when it is empty and does not produce), and the elements in the pool.

``` c
void pool_clear (Pool *pool);
void pool_reset (Pool *pool);
```

Forget the elements without destroying them, or destroy them, keeping the pool.

## Types

| Type | What it is |
|---|---|
| `Pool` | `elements`, `count`, `capacity`, `max_elements`, `mutex`, `destroy`, `create`, `produce` |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `POOL_DEFAULT_CAPACITY` | 16 | The room of a new pool, it doubles |
| `POOL_NO_LIMIT` | 0 | Keep every element |

## See also

- [[Freelists]]: a pool that never allocates, for the hot objects of cerver
- [[Performance]]: how cerver recycles its objects
