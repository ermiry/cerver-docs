# Lists

`#include <cerver/collections/dlist.h>`

A `DoubleList` is a doubly linked list of pointers, with a destroy function for its data, a comparator, and a mutex. Cerver keeps the connections of a client, the lobbies and players of a game, the admins and the game types in lists, and returns lists from the file helpers.

## Concepts

- **Thread safety:** the functions lock the mutex of the list, except the ones that end in `_unsafe`, for a caller that already holds it (`dlist->mutex`) or a list that only one thread uses. The macros (`dlist_for_each ()`) do not lock.
- **Ownership:** `dlist_delete ()` and `dlist_reset ()` release the data with the `destroy` of the list (`free ()` when it is `NULL`). `dlist_clear ()` only releases the elements and keeps the data, for data that other structures also point to.
- **The comparator** returns a negative number, 0 or a positive number, like `strcmp ()`; the searches, the removals by query, `dlist_insert_in_order ()` and `dlist_sort ()` use it, or one passed to the call.

## Usage

``` c
DoubleList *names = dlist_init (str_delete, str_comparator);

(void) dlist_insert_at_end (names, str_new ("cerver"));
(void) dlist_insert_at_start (names, str_new ("ermiry"));

ListElement *le = NULL;
dlist_for_each (names, le) {
	const String *name = (const String *) le->data;
	cerver_log_msg ("%s", name->str);
}

String *query = str_new ("cerver");
String *found = (String *) dlist_remove (names, query, NULL);
str_delete (found);
str_delete (query);

dlist_delete (names);
```

## Reference

### Create and delete

``` c
DoubleList *dlist_init (void (*destroy)(void *data), int (*compare)(const void *one, const void *two));
void dlist_set_compare (DoubleList *list, int (*compare)(const void *one, const void *two));
void dlist_set_destroy (DoubleList *list, void (*destroy)(void *data));

void dlist_delete (void *dlist_ptr);
int dlist_delete_if_empty (void *dlist_ptr);
int dlist_delete_if_not_empty (void *dlist_ptr);

void dlist_reset (DoubleList *dlist);
void dlist_clear (void *dlist_ptr);
void dlist_clear_and_delete (void *dlist_ptr);
void dlist_clear_or_delete (void *dlist_ptr);
```

- `dlist_init ()` creates a list; `destroy` `NULL` uses `free ()`.
- `dlist_delete ()` releases the list, its elements and their data; the `_if_empty` and `_if_not_empty` versions only do it in that case, and return 0 when they deleted it.
- `dlist_reset ()` releases the elements and their data and keeps the list; `dlist_clear ()` releases the elements and keeps the data; `dlist_clear_and_delete ()` clears and deletes the list; `dlist_clear_or_delete ()` clears a list that has elements and deletes an empty one.

### Size

``` c
size_t dlist_size (const DoubleList *dlist);
bool dlist_is_empty (const DoubleList *dlist);
bool dlist_is_not_empty (const DoubleList *dlist);
```

### Insert

``` c
int dlist_insert_before (DoubleList *dlist, ListElement *element, const void *data);
int dlist_insert_after (DoubleList *dlist, ListElement *element, const void *data);
int dlist_insert_at (DoubleList *dlist, const void *data, const unsigned int pos);
int dlist_insert_at_start (DoubleList *dlist, const void *data);
int dlist_insert_at_end (DoubleList *dlist, const void *data);
int dlist_insert_in_order (DoubleList *dlist, const void *data);
```

Insert before or after an element (`NULL` is the start of the list), at a position (0 indexed; past the end inserts at the end), at the start, at the end, or in order with the comparator. All return 0 on success, 1 on error. These do it without the mutex:

``` c
int dlist_insert_before_unsafe (DoubleList *dlist, ListElement *element, const void *data);
int dlist_insert_after_unsafe (DoubleList *dlist, ListElement *element, const void *data);
int dlist_insert_at_start_unsafe (DoubleList *dlist, const void *data);
int dlist_insert_at_end_unsafe (DoubleList *dlist, const void *data);
```

### Remove

``` c
void *dlist_remove (DoubleList *dlist, const void *query, int (*compare)(const void *one, const void *two));
void *dlist_remove_element (DoubleList *dlist, ListElement *element);
void *dlist_remove_start (DoubleList *dlist);
void *dlist_remove_end (DoubleList *dlist);
void *dlist_remove_at (DoubleList *dlist, const unsigned int idx);

unsigned int dlist_remove_by_condition (
	DoubleList *dlist,
	bool (*compare)(const void *one, const void *two), const void *match,
	bool delete_data
);
```

Remove the element that compares equal to `query` (with `compare`, or the comparator of the list), an element (`NULL` is the start), the first, the last, or the one at an index, and return its data, that the caller now owns (or `NULL`). These do it without the mutex:

``` c
void *dlist_remove_element_unsafe (DoubleList *dlist, ListElement *element);
void *dlist_remove_start_unsafe (DoubleList *dlist);
void *dlist_remove_end_unsafe (DoubleList *dlist);
```

`dlist_remove_by_condition ()` removes every element for which `compare (data, match)` is `true`, releasing their data with `delete_data`, and returns how many it removed.

### Search and traverse

``` c
void *dlist_search (const DoubleList *dlist, const void *data, int (*compare)(const void *one, const void *two));
ListElement *dlist_get_element (const DoubleList *dlist, const void *data, int (*compare)(const void *one, const void *two));
ListElement *dlist_get_element_at (const DoubleList *dlist, const unsigned int idx);
void *dlist_get_at (const DoubleList *dlist, const unsigned int idx);

int dlist_traverse (
	const DoubleList *dlist,
	void (*method)(void *list_element_data, void *method_args), void *method_args
);
```

Find the data or the element that compares equal to `data`, get the element or the data at an index, and call `method` with the data of each element and `method_args`, with the mutex (0 on success, 1 on error).

### Sort, copy, split and merge

``` c
int dlist_sort (DoubleList *dlist, int (*compare)(const void *one, const void *two));
void **dlist_to_array (const DoubleList *dlist, size_t *count);
DoubleList *dlist_copy (const DoubleList *dlist);
DoubleList *dlist_clone (const DoubleList *dlist, void *(*clone) (const void *original));
DoubleList *dlist_split_half (DoubleList *dlist);
DoubleList *dlist_split_by_condition (DoubleList *dlist, bool (*compare)(const void *one, const void *two), const void *match);
void dlist_merge_two (DoubleList *one, DoubleList *two);
DoubleList *dlist_merge_two_by_condition (DoubleList *one, DoubleList *two, bool (*compare)(const void *one, const void *two), const void *match);
DoubleList *dlist_merge_many (DoubleList *many_dlists);
```

- `dlist_sort ()` sorts with merge sort (0 on success, 1 on error).
- `dlist_to_array ()` returns a new array of the pointers, and their count; the caller releases the array, not the data.
- `dlist_copy ()` returns a list with the same pointers (clear one of the two before deleting both); `dlist_clone ()` creates new data with `clone`.
- `dlist_split_half ()` moves the second half to a new list (the first one keeps the extra element of an odd size); `dlist_split_by_condition ()` moves the matching elements to a new list.
- `dlist_merge_two ()` moves the elements of `two` to the end of `one`; `dlist_merge_two_by_condition ()` moves the matching elements of both to a new list; `dlist_merge_many ()` moves the elements of a list of lists to a new list.

### Macros

| Macro | What it is |
|---|---|
| `dlist_start (list)`, `dlist_end (list)` | The first and the last element |
| `dlist_element_data (element)`, `dlist_element_next (element)` | The data and the next element |
| `dlist_for_each (dlist, le)`, `dlist_for_each_backwards (dlist, le)` | A `for` over the elements, without the mutex |

## Types

| Type | What it is |
|---|---|
| `DoubleList` | `size`, `start`, `end`, `destroy`, `compare`, `mutex` |
| `ListElement` | `prev`, `data`, `next` |

## See also

- [[Queues]]: a queue on top of a list
- [[Trees]] and [[Hashtables]]: the other collections
