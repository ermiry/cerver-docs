# Base64

`#include <cerver/utils/base64.h>`

The standard base64 (RFC 4648) encoder and decoder of cerver, with an AVX2 version that is used when the library is built for a processor that has it.

## Usage

``` c
const char *input = "cerver";
const size_t input_size = strlen (input);

char encoded[base64_encode_len (6)];
(void) base64_encode (encoded, input, input_size);

char decoded[base64_decode_len (sizeof (encoded))];
size_t decoded_size = base64_decode (decoded, encoded, strlen (encoded));
if (decoded_size == (size_t) -1) {
	// not valid base64
}
```

## Reference

#### `base64_encode ()`

``` c
size_t base64_encode (char *encoded, const char *input, size_t inlen);
```

Encodes `inlen` bytes into `encoded`, a buffer of at least `base64_encode_len (inlen)` bytes, terminated with a NUL.

- **Returns** the length of the encoded text plus the NUL.

#### `base64_decode ()`

``` c
size_t base64_decode (char *output, const char *bufcoded, size_t codedlen);
```

Decodes `codedlen` characters of base64 into `output`, a buffer of at least `base64_decode_len (codedlen)` bytes. Any character that is not base64, like a space, is an error.

- **Returns** the bytes decoded, or `(size_t) -1` on error.

### Macros

| Macro | What it is |
|---|---|
| `base64_encode_len (A)` | The bytes of the buffer to encode `A` bytes, with the NUL |
| `base64_encode_strlen (A)` | The length of the encoded text of `A` bytes |
| `base64_decode_len (A)` | The bytes of the buffer to decode `A` characters |

## See also

- [[SHA256]] and [[Utilities]]
