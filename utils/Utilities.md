# Utilities

`#include <cerver/utils/utils.h>`

Small helpers that cerver uses and exports: numbers, random values, conversions and plain C strings. For strings that grow, see [[Strings]].

## Reference

### Numbers

``` c
int clamp_int (const int val, const int min, const int max);
int abs_int (const int value);
float lerp (const float first, const float second, const float by);
bool float_compare (const float f1, const float f2);
bool system_is_little_endian (void);
```

Clamp a value to a range, its absolute value, the linear interpolation between two values, whether two floats differ by less than 0.00001, and whether the processor is little endian.

### Random

``` c
void random_set_seed (const unsigned int seed);
int random_int_in_range (const int min, int max);
float random_float (const float abs);
```

Seed and use the pseudo random generator of the C library. They are **not** suitable for tokens, ids or anything that must not be guessed: cerver uses `getrandom ()` for those, like the session ids (see [[Sessions]]).

### Conversions

``` c
int xtoi (char *hex_string);
char *itoa (int i, char *b);
bool bool_value_from_string (const char *string_value);
```

A hex text to an `int`, an `int` to text in a buffer that the caller gives, and `"true"` (any case) to `true`.

### C strings

``` c
void c_string_copy (char *to, const char *from);
void c_string_n_copy (char *to, const char *from, size_t n);
char *c_string_concat (const char *s1, const char *s2, size_t *des_size);
size_t c_string_concat_safe (const char *s1, const char *s2, char *des, size_t des_size);
char *c_string_create (const char *format, ...);
```

Copy a C string into a buffer, copy at most `n` bytes, concatenate two into a new buffer (with its size in `des_size`), concatenate two into a buffer of `des_size` bytes only if they fit (returns the length, 0 if they did not fit), and create one from a format like `printf ()`. The new strings are released with `free ()`.

``` c
void c_string_remove_spaces (char *s);
void c_string_remove_line_breaks (char *s);
void c_string_remove_spaces_and_line_breaks (char *s);
void c_string_remove_char (char *string, char garbage);
char *c_string_trim (char *str);
char *c_string_strip_quotes (char *str);
```

Remove, in place, the spaces, the line breaks, both, or every appearance of a character; trim the white space at both ends, and strip the quotes.

``` c
size_t c_string_count_tokens (const char *original, const char delim);
char **c_string_split (const char *original, const char delim, size_t *n_tokens);
char *c_string_reverse (const char *str);
bool c_string_starts_with (const char *str, const char *substr);
char *c_string_find_sub_in_len (const char *haystack, const char *needle, size_t len);
char *c_string_create_with_ptrs (char *first, char *last);
```

Count the tokens of a delimiter, split into a new array of new strings (thread safe, with `strtok_r ()`), a reversed copy, whether a string starts with another one, find a substring in the first `len` bytes, and a new string with the bytes between two pointers of the same string.

``` c
char *c_string_remove_sub (char *str, const char *sub);
char *c_string_remove_sub_after_token (char *str, const char token, char **sub);
char *c_string_remove_sub_after_token_with_idx (char *str, const char token, char **sub, int idx);
char *c_string_remove_sub_simetric_token (char *str, const char token, char **sub);
char *c_string_remove_sub_range_token (char *str, const char token, int first, int last, char **sub);
```

Return a new string without a part of the original, and the part in `sub` when it is not `NULL`:

- `c_string_remove_sub ()`: without the exact substring;
- `c_string_remove_sub_after_token ()`: without what follows the first `token`;
- `c_string_remove_sub_after_token_with_idx ()`: without what follows the appearance `idx` of `token`, -1 for the last one (`/home/user/Documents` with `'/'` and -1 gives `Documents` as `sub`);
- `c_string_remove_sub_simetric_token ()`: without what is between the first and the last `token`;
- `c_string_remove_sub_range_token ()`: without what is between the appearances `first` and `last` of `token`.

## See also

- [[Strings]], [[Base64]], [[SHA256]]
