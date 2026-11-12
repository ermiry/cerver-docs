# Base64

`#include <cerver/utils/base64.h>`

The standard base64 (RFC 4648) encoder and decoder of cerver, with an AVX2 version that is used when the library is built for a processor that has it, and a canonical base64url (RFC 4648 §5) encoder and decoder.

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

``` c
const unsigned char bytes[] = { 0xfb, 0xff };

char url[base64url_encode_length (sizeof (bytes)) + 1];
(void) base64url_encode (url, bytes, sizeof (bytes));    // "-_8"

unsigned char back[base64url_decode_length (3)];
size_t back_length = 0;
if (base64url_decode (back, &back_length, url, strlen (url))) {
	// not canonical base64url
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

#### `base64url_encode ()`

``` c
size_t base64url_encode (char *encoded, const unsigned char *input, size_t input_length);
```

Encodes `input_length` bytes as base64url, with `-` and `_` and without padding, into `encoded`, a buffer of at least `base64url_encode_length (input_length) + 1` bytes, terminated with a NUL.

- **Returns** the length of the encoded text, without the NUL.

#### `base64url_decode ()`

``` c
unsigned int base64url_decode (
	unsigned char *output, size_t *output_length,
	const char *encoded, size_t encoded_length
);
```

Decodes `encoded_length` characters of canonical base64url into `output`, a buffer of at least `base64url_decode_length (encoded_length)` bytes, and sets `output_length` to the bytes decoded. Only the characters of the alphabet are valid, without padding, and the unused bits of the last character must be zero, so every sequence of bytes has a single valid form: `Zg` decodes, `Zh` and `Zg==` are errors.

- **Returns** 0 on success, 1 on error, with `output_length` set to 0.

### Macros

| Macro | What it is |
|---|---|
| `base64_encode_len (A)` | The bytes of the buffer to encode `A` bytes, with the NUL |
| `base64_encode_strlen (A)` | The length of the encoded text of `A` bytes |
| `base64_decode_len (A)` | The bytes of the buffer to decode `A` characters |
| `base64url_encode_length (length)` | The characters of the base64url text of `length` bytes, without the NUL |
| `base64url_decode_length (length)` | The bytes that `length` characters of base64url can give |

Every macro takes an expression as its argument: `base64_encode_len (size + 1)` is the length of `size + 1` bytes.

## See also

- [[SHA256]] and [[Utilities]]
