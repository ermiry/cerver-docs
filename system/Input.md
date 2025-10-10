# Input

`#include <cerver/input.h>`

Reading from the terminal, for the interactive programs of the examples.

## Reference

``` c
void input_clean_stdin (void);
char *input_get_line (void);
unsigned int input_password (char *password);
```

- `input_clean_stdin ()` discards what is left in `stdin` up to the next line break.
- `input_get_line ()` reads a line, without its line break, into a new string released with `free ()`.
- `input_password ()` reads a password without echoing it, into a buffer of `INPUT_PASSWORD_SIZE` (129) bytes, of up to 128 characters (0 on success, 1 on error).

## Constants

| Constant | Value |
|---|---|
| `INPUT_PASSWORD_SIZE` | 129 |
| `INPUT_PASSWORD_SCAN_FORMAT` | `"%128s"` |

## See also

- [[Examples]]: the interactive clients
