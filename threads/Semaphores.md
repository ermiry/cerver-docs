# Semaphores

`#include <cerver/threads/bsem.h>`

A binary semaphore, `bsem`: a value of 0 or 1 with a mutex and a condition. A thread waits until it is 1 and takes it back to 0; another one posts it to 1 to wake one or every waiting thread.

## Usage

``` c
bsem *ready = bsem_new ();
bsem_init (ready, 0);

// a thread waits
bsem_wait (ready);

// another one wakes it up
bsem_post (ready);

bsem_delete (ready);
```

## Reference

#### `bsem_new ()`, `bsem_delete ()`

``` c
bsem *bsem_new (void);
void bsem_delete (void *bsem_ptr);
```

Allocate a semaphore with its mutex and condition, and destroy and release it.

#### `bsem_init ()`, `bsem_reset ()`

``` c
void bsem_init (bsem *bsem_p, int value);
void bsem_reset (bsem *bsem_p);
```

Set the value to 1 or 0, and back to 0, keeping its mutex and condition.

#### `bsem_post ()`, `bsem_post_all ()`

``` c
void bsem_post (bsem *bsem_p);
void bsem_post_all (bsem *bsem_p);
```

Set the value to 1 and wake up one waiting thread, or every one.

#### `bsem_wait ()`

``` c
void bsem_wait (bsem *bsem_p);
```

Waits until the value is 1, and sets it back to 0.

## Types

| Type | What it is |
|---|---|
| `bsem` | `mutex`, `cond` and the value `v` |

## See also

- [[Threads]]: the mutexes and conditions of cerver
