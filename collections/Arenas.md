# Arenas

`#include <cerver/collections/arena.h>`

An `Arena` is a bump allocator: it hands out memory from chunks, and releases all of it at once. It is fast for many small allocations that live and die together, like the fields of an object that the [[Serializer]] decodes. An arena is **not thread safe**: every thread uses its own.

## Concepts

- Each allocation takes the next bytes of the current chunk, aligned to `ARENA_ALIGNMENT` (16). An allocation that does not fit in a chunk is a **large allocation** of its own.
- `arena_reset ()` releases every allocation at once: the chunks are kept to be reused, and the large allocations are freed. Nothing is released one by one.

## Usage

``` c
Arena *arena = arena_create (0);

MyObject object;
if (serializer_decode (&my_object_schema, buffer, size, arena, &object) == SERIALIZER_RESULT_NONE) {
	// the arena fields of object point into the arena
}

arena_reset (arena);
arena_delete (arena);
```

## Reference

``` c
Arena *arena_create (const size_t chunk_size);
void arena_delete (Arena *arena);
```

Create an arena with chunks of `chunk_size` bytes (0 uses `ARENA_DEFAULT_CHUNK_SIZE`, 16 KiB), and release it with every chunk and large allocation.

``` c
void *arena_alloc (Arena *arena, const size_t size);
void *arena_calloc (Arena *arena, const size_t size);
```

Return `size` bytes aligned to `ARENA_ALIGNMENT`, not initialized or set to 0, or `NULL` on error.

``` c
void arena_reset (Arena *arena);
```

Releases every allocation, keeping the chunks.

## Types

| Type | What it is |
|---|---|
| `Arena` | `chunk_size`, its chunks, its large allocations, the bytes `used` and `n_allocations` |

## Constants

| Constant | Value |
|---|---|
| `ARENA_DEFAULT_CHUNK_SIZE` | 16384 |
| `ARENA_ALIGNMENT` | 16 |

## See also

- [[Serializer]]: the arena fields
