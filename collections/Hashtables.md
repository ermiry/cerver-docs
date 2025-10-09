# Hashtables

`#include <cerver/collections/htab.h>`

An `Htab` maps keys of any size to values, with buckets of chained nodes, a hash function (FNV-1a by default) and a mutex. A cerver finds its clients by the fd of their sockets with one, and the scoreboards keep their scores in them.

## Concepts

- **Keys** are copied when they are inserted, with `key_create` or a generic copy of `key_size` bytes, and released with `key_delete` or `free ()`. They are compared with `key_compare` or byte by byte.
- **Values** are pointers that the table keeps; `htab_destroy ()` releases them with `delete_data`, and `htab_remove ()` gives them back to the caller.
- **Size:** the number of buckets is rounded up to a power of 2 (`HTAB_DEFAULT_INIT_SIZE`, 32, for 0), and it doubles when the table is 3/4 full, except with a custom `key_create`.
- The functions lock the mutex of the table.

## Usage

``` c
Htab *sessions = htab_create (0, NULL, free);

const char *key = "session-one";
(void) htab_insert (sessions, key, strlen (key), strdup ("value"), sizeof (char *));

char *value = (char *) htab_get (sessions, key, strlen (key));

htab_destroy (sessions);
```

## Reference

``` c
Htab *htab_create (
	size_t size,
	size_t (*hash)(const void *key, size_t key_size, size_t table_size),
	void (*delete_data)(void *data)
);
void htab_set_key_create (Htab *htab, void *(*key_create)(const void *));
void htab_set_key_delete (Htab *htab, void (*key_delete)(void *));
void htab_set_key_comparator (Htab *htab, int (*key_compare)(const void *one, const void *two));
void htab_destroy (Htab *ht);
```

Create a table with its buckets, its hash (`NULL` for FNV-1a) and how its values are released (`NULL` does not release them); set how its keys are created, released and compared; destroy it with its keys and values.

``` c
size_t htab_size (Htab *htab);
bool htab_is_empty (Htab *htab);
bool htab_is_not_empty (Htab *htab);
bool htab_contains_key (Htab *ht, const void *key, size_t key_size);
```

How many values the table has, and whether a key is in it.

``` c
int htab_insert (Htab *ht, const void *key, size_t key_size, void *val, size_t val_size);
void *htab_get (Htab *ht, const void *key, size_t key_size);
void *htab_remove (Htab *ht, const void *key, size_t key_size);
```

Insert a value with its key (0 on success, 1 on error or if the key exists), get the value of a key (`NULL` without one), and remove it and return it to the caller.

``` c
void htab_print (Htab *htab);
```

Prints the buckets and their nodes, for a table of `int` keys and values; it is for debugging.

## Types

| Type | What it is |
|---|---|
| `Htab` | `table` (the buckets), `size`, `count`, `hash`, the key functions, `delete_data`, `mutex` |
| `HtabBucket` | `start`, `count` |
| `HtabNode` | `next`, `key`, `key_size`, `val`, `val_size` |

## Constants

| Constant | Value |
|---|---|
| `HTAB_DEFAULT_INIT_SIZE` | 32 |

## See also

- [[Lists]] and [[Trees]]
