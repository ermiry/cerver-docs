# Trees

`#include <cerver/collections/avl.h>`

An `AVLTree` is a balanced binary search tree of pointers, ordered by a comparator, with a mutex. A cerver keeps its clients and its on hold connections in trees.

## Concepts

- The **comparator** orders the data: a negative number, 0 or a positive number. The same data, or a query with the fields that the comparator reads, finds a node.
- The **destroy** function releases the data of the nodes when the tree is cleared or deleted; without one the data stays, and the caller releases it.
- `avl_insert_node ()` and `avl_remove_node ()` lock the mutex of the tree (`tree->mutex`); `avl_get_node_data ()` does not, and `avl_get_node_data_safe ()` does. Walking the nodes from `root`, like `client_broadcast_to_all_avl ()`, needs the mutex taken by the caller.

## Usage

``` c
AVLTree *tree = avl_init (my_comparator, my_delete);

(void) avl_insert_node (tree, my_item);

MyItem query = { .id = 7 };
MyItem *found = (MyItem *) avl_get_node_data (tree, &query, NULL);

MyItem *removed = (MyItem *) avl_remove_node (tree, &query);
my_delete (removed);

avl_delete (tree);
```

## Reference

``` c
AVLTree *avl_init (Comparator comparator, void (*destroy)(void *data));
void avl_set_comparator (AVLTree *tree, Comparator comparator);
void avl_set_destroy (AVLTree *tree, void (*destroy)(void *data));
void avl_delete (AVLTree *tree);
```

Create a tree, set its comparator and its destroy, and delete it with its nodes (the data is released only with a destroy).

``` c
bool avl_is_empty (AVLTree *tree);
bool avl_is_not_empty (AVLTree *tree);
size_t avl_size (const AVLTree *avl);
```

Whether the tree has nodes, and how many.

``` c
unsigned int avl_insert_node (AVLTree *tree, void *data);
void *avl_remove_node (AVLTree *tree, void *data);
unsigned int avl_clear_tree (AVLTree *tree, void (*destroy)(void *data));
```

Insert data (0 on success, 1 on error), remove the node of some data and return it (or `NULL`), and remove every node, releasing the data with `destroy` or the one of the tree (0 on success, 1 on error).

``` c
void *avl_get_node_data (AVLTree *tree, void *id, Comparator comparator);
void *avl_get_node_data_safe (AVLTree *tree, void *id, Comparator comparator);
```

Return the data that compares equal to `id`, with the comparator of the tree or another one, without and with the mutex.

## Types

| Type | What it is |
|---|---|
| `AVLTree` | `root`, `size`, `comparator`, `destroy`, `mutex` |
| `AVLNode` | `id` (the data), `balance`, `left`, `right` |
| `Comparator` | `int (*) (const void *, const void *)` (see [[Types]]) |

## See also

- [[Lists]] and [[Hashtables]]
