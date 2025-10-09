# Strings

`#include <cerver/types/string.h>`

A `String` is a C string that knows its length and its capacity, so appends grow it geometrically instead of reallocating each time. It is always terminated with a NUL, and it can hold bytes with NUL characters inside (`str_create_from_bytes ()`, `str_append_bytes ()`). Cerver uses it for names, messages, tokens and paths.

## Concepts

- `len` is the bytes without the NUL; `max_len` is the bytes allocated for `str`, with the NUL, or 0 when they are exactly `len + 1`. Code that changes `str` by hand sets `max_len` to 0.
- A string of an arena (`str_create_from_bytes_arena ()`) lives with the arena: it is not deleted with `str_delete ()` nor grown.
- The strings are not thread safe.

## Usage

``` c
String *name = str_new ("cerver");
str_append_c_string (name, " 1.8");

String *message = str_create ("%s has %u clients", name->str, 4u);

(void) str_reserve (message, 256);
(void) str_append_bytes (message, "\0binary", 7);

str_delete (message);
str_delete (name);
```

## Reference

### Create and delete

``` c
String *str_new (const char *str);
String *str_create (const char *format, ...);
String *str_allocate (const unsigned int max_len);
String *str_create_from_bytes (const char *bytes, const size_t n);
String *str_create_from_bytes_arena (struct Arena *arena, const char *bytes, const size_t n);
String *str_clone (const String *original);
String *str_concat (const String *s1, const String *s2);
void str_delete (void *str_ptr);
```

Create a string from a C string (`NULL` gives one without text, with `str` in `NULL`), from a format like `printf ()`, empty with room for `max_len` bytes, from exactly `n` bytes (in the heap or in an arena), as a copy, or as the concatenation of two; and delete one.

### Change

``` c
void str_set (String *str, const char *format, ...);
void str_set_with_args (String *str, const char *format, va_list args);
void str_replace (String *str, const char *format, ...);
void str_replace_with (String *str, const char *c_str);
void str_n_replace_with (String *str, const char *c_str, const size_t n);
void str_copy (String *to, const String *from);
void str_n_copy (String *to, const String *from, const size_t n);
```

Replace the contents with a format, a C string, its first `n` bytes, or another string.

### Grow

``` c
int str_reserve (String *str, const size_t capacity);
int str_append_bytes (String *str, const char *bytes, const size_t n);
void str_append_char (String *str, const char c);
void str_append_c_string (String *str, const char *c_str);
void str_append_n_from_c_string (String *str, const char *c_str, const size_t n);
```

- `str_reserve ()` makes sure that the string has at least `capacity` bytes, with the NUL, so the appends that fit do not reallocate; it never shrinks it (0 on success, 1 on error, keeping the string).
- `str_append_bytes ()` appends exactly `n` bytes, that can contain NULs, growing geometrically (0 on success, 1 on error, keeping the string).
- The others append a character, a C string or its first `n` characters.

### Compare and search

``` c
int str_compare (const String *s1, const String *s2);
int str_comparator (const void *a, const void *b);
int str_contains (const String *str, const char *to_find);
```

Compare two strings like `strcmp ()`, as a comparator for the collections, and look for a C string: 0 for an exact match, 1 when the letters match but not the length, -1 without a match.

### Transform

``` c
void str_to_upper (String *string);
void str_to_lower (String *string);
void str_remove_char (String *string, const char garbage);
void str_remove_last_char (String *string);
char **str_split (String *string, const char delim, int *n_tokens);
```

Change the case, remove every appearance of a character or the last character, and split by a delimiter into a new array of new C strings, with their count; the caller releases each one and the array with `free ()`.

## Types

| Type | What it is |
|---|---|
| `String` | `max_len`, `len`, `str` |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `STRING_MIN_CAPACITY` | 16 | The least bytes of a string that grows |
| `STRING_GROWTH_FACTOR` | 2 | |

## See also

- [[Utilities]]: the helpers for plain C strings
- [[Arenas]]: the strings of an arena
