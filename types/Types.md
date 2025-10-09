# Types

`#include <cerver/types/types.h>`

The integer types and the function types that the whole API uses.

## Integers

| Type | What it is |
|---|---|
| `u8`, `u16`, `u32`, `u64` | `uint8_t` … `uint64_t` |
| `i8`, `i16`, `i32`, `i64` | `int8_t` … `int64_t` |
| `uchar`, `ascii_char` | `unsigned char` |

## Functions

| Type | Signature | Used by |
|---|---|---|
| `Action` | `void (*) (void *)` | Handlers, updates, delete functions |
| `Work` | `void *(*) (void *)` | Events and errors, threads |
| `delegate` | `u8 (*) (void *)` | The authenticate methods: 0 accepts |
| `Func` | `u8 (*) (void)` | |
| `Comparator` | `int (*) (const void *, const void *)` | Collections, lobbies and players |

## Values

`cerver_type_t` names kinds of values: `CERVER_TYPE_UNDEFINED`, `DOUBLE`, `CHAR`, `UTF8`, `ARRAY`, `BOOL`, `DATE_TIME`, `NULL`, `INT32`, `TIMESTAMP` and `INT64`. It is not the `CerverType` of a cerver (`CERVER_TYPE_CUSTOM`…), that only shares its prefix. The objects that cerver sends use the [[Serializer]].

## See also

- [[Strings]]: the `String` of cerver
- [[Configuration]]: the macros of `config.h`
