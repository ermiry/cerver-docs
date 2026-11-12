# Version

`#include <cerver/version.h>`

The version of cerver that a program was built with.

## Reference

| Macro | What it is |
|---|---|
| `CERVER_VERSION` | The version, like `"1.8.1"`; the same as `version.txt` |
| `CERVER_VERSION_NAME` | Its name, like `"Release 1.8.1"` or `"Beta 1.9b-1"` |
| `CERVER_VERSION_DATE` | The date of the version, `day/month/year` |
| `CERVER_VERSION_TIME` | Its time |
| `CERVER_VERSION_AUTHOR` | Its author |

``` c
void cerver_version_print_full (void);
void cerver_version_print_version_id (void);
void cerver_version_print_version_name (void);
```

Print every value with the log, the version, or its name.

## See also

- [[Migration]]: the changes between versions
