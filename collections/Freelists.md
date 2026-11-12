# Freelists

`#include <cerver/collections/freelist.h>`

A `FreeList` is a bounded stack of released elements that **never allocates**: each element has a member that points to the next one, so the list links the elements themselves. Cerver recycles its connections, sockets, clients and receives with free lists (see [[Performance]]).

## Concepts

- The list is declared statically with `FREELIST_INITIALIZER (type, member, limit)`, where `member` is the pointer to the next element inside `type`.
- `freelist_push ()` keeps a released element while the list has room (`limit`); a full list returns 1 and the caller releases the element.
- A build with a sanitizer defines `CERVER_RECYCLE_DISABLED`, and `FREELIST_RECYCLE_LIMIT (limit)` is 0, so nothing is recycled and a use after free is reported instead of being hidden.
- The functions lock the mutex of the list, so any thread can use it.

## Usage

``` c
typedef struct MyItem {

	int value;
	struct MyItem *recycle_next;

} MyItem;

static FreeList items = FREELIST_INITIALIZER (MyItem, recycle_next, FREELIST_RECYCLE_LIMIT (FREELIST_DEFAULT_LIMIT));

static MyItem *my_item_new (void) {

	MyItem *item = (MyItem *) freelist_pop (&items);
	return item ? item : (MyItem *) malloc (sizeof (MyItem));

}

static void my_item_delete (MyItem *item) {

	if (freelist_push (&items, item)) free (item);

}
```

## Reference

``` c
void *freelist_pop (FreeList *list);
unsigned int freelist_push (FreeList *list, void *element);
unsigned int freelist_size (FreeList *list);
void freelist_drain (FreeList *list, void (*destroy) (void *element));
```

Take a released element (`NULL` when it is empty), keep one (0 if it was kept, 1 if the list is full), the elements in the list, and release every element with `destroy`.

## Types

| Type | What it is |
|---|---|
| `FreeList` | `mutex`, `head`, `count`, `limit`, `next_offset` |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `FREELIST_DEFAULT_LIMIT` | 1024 | The elements that a list keeps by default |
| `FREELIST_INITIALIZER (type, member, limit)` | | A static free list |
| `FREELIST_RECYCLE_LIMIT (limit)` | `limit`, or 0 with sanitizers | |

## See also

- [[Pools]]: a pool on an array, for any element
- [[Performance]]: the objects that cerver recycles
