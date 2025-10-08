# Serializer

`#include <cerver/serializer.h>`

The serializer encodes C structs as **CBOR** maps (RFC 8949) and decodes them back with a **schema**: a table of the fields of the struct, generated from a single list. It is how cerver sends its own objects (its info, the session tokens, the errors, the file headers, the lobbies), and how an application sends its own in the data of a packet (`packet_create_serialized ()`, `packet_get_object ()`, see [[Packets]]).

## Concepts

### Why CBOR maps with field ids

Each field of an object has an **id**, from 1 to `SERIALIZER_FIELDS_MAX` (64), and the object is a map from ids to values, with the version of its schema in the key 0. So:

- **any CBOR library can read it**, in any language;
- **a peer ignores the fields that it does not know**, and gives their default to the ones that are missing, so a schema can get new fields without breaking the older peers;
- **the version of the schema** only changes with a change that breaks the older peers, and a decoder rejects an object of another version (`SERIALIZER_RESULT_VERSION`).

### Untrusted input

The decoder is written for data from the network. It validates the whole object before it writes anything (two passes), and it rejects: a buffer that ends in the middle of an item, an object bigger than its maximum size (64 KiB), nested deeper than its maximum depth (8), or with more elements than its maximum (1024); a value of another type, a number that does not fit its field, a text, bytes or array longer than its field, a text that is not UTF-8 or that has a NUL, a repeated field, a missing required field, a key that is not a field id, a feature of CBOR that cerver does not take, and bytes after the object. Nothing is allocated, except the fields that take memory from an **arena**.

### Field kinds

| Kind | Member of the struct | Notes |
|---|---|---|
| `BOOL`, `U8`, `U16`, `U32`, `U64`, `I8`, `I16`, `I32`, `I64`, `F32`, `F64` | The C type | A default for when it is missing |
| `ENUM` | An `i32` | With a default, a minimum and a maximum |
| `TEXT` | `char name[capacity]` | The capacity counts the NUL |
| `BYTES` | `u8 name[capacity]` and `u32 name_count` | |
| `TEXT_ARENA`, `BYTES_ARENA` | `char *name`, or `u8 *name` and `u32 name_count` | The memory comes from the `Arena` of the decode; the capacity is the maximum length |
| `ARRAY` | `kind name[capacity]` and `u32 name_count` | An array of scalars |
| `OBJECT` | `Type name` | A nested object, with its own schema |
| `OBJECTS` | `Type name[capacity]` and `u32 name_count` | An array of objects |

## Usage

A schema is described once with a list of fields:

``` c
#define MY_USER_NAME_SIZE		64
#define MY_USER_ROLES			8
#define MY_USER_VERSION			1

// SCALAR (T, id, kind, name, required, default)
// ENUM (T, id, name, required, default, minimum, maximum)
// STRING (T, id, kind, name, capacity, required)
// ARRAY (T, id, kind, name, capacity, required)
// OBJECT (T, id, Type, schema, name, required)
// OBJECTS (T, id, Type, schema, name, capacity, required)
#define MY_USER_FIELDS(T, SCALAR, ENUM, STRING, ARRAY, OBJECT, OBJECTS)		\
	SCALAR(T, 1, U64, id, SERIALIZER_REQUIRED, 0)							\
	STRING(T, 2, TEXT, name, MY_USER_NAME_SIZE, SERIALIZER_REQUIRED)		\
	ENUM(T, 3, level, SERIALIZER_OPTIONAL, 0, 0, 3)							\
	ARRAY(T, 4, U16, roles, MY_USER_ROLES, SERIALIZER_OPTIONAL)

// in a header: the struct MyUser and the declaration of my_user_schema
SERIALIZER_DECLARE (MyUser, my_user, MY_USER_FIELDS)

// in a source: the table of the fields and the schema
SERIALIZER_DEFINE (MyUser, my_user, MY_USER_VERSION, MY_USER_FIELDS)
```

Encoding and decoding:

``` c
MyUser user;
serializer_object_init (&my_user_schema, &user);
user.id = 7;
(void) strncpy (user.name, "ermiry", sizeof (user.name) - 1);

u8 buffer[SERIALIZER_DEFAULT_MAX_SIZE];
size_t written = 0;
if (serializer_encode (&my_user_schema, &user, buffer, sizeof (buffer), &written) == SERIALIZER_RESULT_NONE) {
	MyUser decoded;
	SerializerResult result = serializer_decode (&my_user_schema, buffer, written, NULL, &decoded);
	if (result != SERIALIZER_RESULT_NONE) {
		cerver_log_error ("%s", serializer_result_string (result));
	}
}
```

With a field of an arena, the decode takes an `Arena` (see [[Arenas]]), whose memory lives until the arena is reset or deleted.

## Reference

#### `serializer_object_init ()`

``` c
void serializer_object_init (const SerializerSchema *schema, void *object);
```

Sets every field of an object to its default: the default of its scalars and enums, empty texts, bytes and arrays, and its nested objects initialized the same way.

#### `serializer_encoded_size ()`

``` c
size_t serializer_encoded_size (const SerializerSchema *schema, const void *object);
```

Returns the bytes of the encoded object, 0 if it can not be encoded.

#### `serializer_encode ()`

``` c
SerializerResult serializer_encode (
	const SerializerSchema *schema, const void *object,
	void *buffer, const size_t size, size_t *written
);
```

Encodes an object in a buffer, without allocating memory. `written` gets the bytes of the encoded object, and can be `NULL`.

- **Returns** `SERIALIZER_RESULT_NONE` on success, or why it failed, like `SERIALIZER_RESULT_BUFFER` for a buffer that is too small.

#### `serializer_decode ()`

``` c
SerializerResult serializer_decode (
	const SerializerSchema *schema,
	const void *buffer, const size_t size,
	Arena *arena, void *object
);
```

Validates and decodes an object that fills the whole buffer. The fields that are missing take their default, and the unknown ones are ignored. The arena gives the memory of the arena fields, and can be `NULL` for a schema without them. The object is only written when the result is `SERIALIZER_RESULT_NONE`.

#### `serializer_result_string ()`, `serializer_type_string ()`

``` c
const char *serializer_result_string (const SerializerResult result);
const char *serializer_type_string (const SerializerType type);
```

The description of a result, like `"A required field is missing"`, and the name of a field kind.

#### `serializer_schemas ()`

``` c
const SerializerSchema *const *serializer_schemas (size_t *count);
```

Returns the schemas of the objects that cerver itself sends, and their count; the fuzzers use it.

### Macros

| Macro | What it does |
|---|---|
| `SERIALIZER_DECLARE (Name, prefix, FIELDS)` | Declares the struct `Name` and `prefix_schema` |
| `SERIALIZER_DEFINE (Name, prefix, version, FIELDS)` | Defines the table of the fields and the schema, with the default limits |
| `SERIALIZER_DEFINE_LIMITS (Name, prefix, version, FIELDS, max_size, max_depth, max_elements)` | The same, with lower limits for that schema |

## Types

| Type | What it is |
|---|---|
| `SerializerSchema` | `name`, `version`, the `size` of the struct, its `fields` and their count, and its limits |
| `SerializerField` | A field: `id`, `type`, `name`, its offset and size in the struct, its capacity, its count member, whether it is `required`, its default, the range of an enum, the type of the elements of an array and the schema of an object |
| `SerializerType` | The kinds of a field, see above |
| `SerializerResult` | `NONE`, `ARGUMENTS`, `TRUNCATED`, `SIZE`, `DEPTH`, `ELEMENTS`, `TYPE`, `RANGE`, `LENGTH`, `UTF8`, `NUL`, `DUPLICATE`, `MISSING`, `VERSION`, `KEY`, `UNSUPPORTED`, `ARENA`, `TRAILING`, `BUFFER` |
| `SerializerDefault` | The default of a field, as an integer, a natural and a number |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `SERIALIZER_DEFAULT_MAX_SIZE` | 65536 | The biggest encoded object |
| `SERIALIZER_DEFAULT_MAX_DEPTH` | 8 | The deepest nesting |
| `SERIALIZER_DEFAULT_MAX_ELEMENTS` | 1024 | The most elements of an array or a map |
| `SERIALIZER_FIELDS_MAX` | 64 | The highest field id |
| `SERIALIZER_VERSION_KEY` | 0 | The key of the version of the schema |
| `SERIALIZER_REQUIRED`, `SERIALIZER_OPTIONAL` | `true`, `false` | |
| `SERIALIZER_TYPE_MAP (XX)`, `SERIALIZER_RESULT_MAP (XX)` | | The X-macros of the kinds and the results |

## See also

- [[Packets]]: `packet_create_serialized ()` and `packet_get_object ()`
- [[Protocol]]: the objects of cerver and their schemas
- [[Arenas]]: the memory of the arena fields
