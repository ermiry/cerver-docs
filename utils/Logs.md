# Logs

`#include <cerver/utils/log.h>`

The log of cerver: messages with a type, colors in a terminal, an optional date and time, and an optional file, written by a thread of its own. `cerver_init ()` starts it and `cerver_end ()` ends it.

## Concepts

- **Types:** every message has a first type, that gives its color and its stream (errors and warnings go to `stderr`), and an optional second type that says where it comes from, like `[ERROR][CERVER]`.
- **Output:** `LOG_OUTPUT_TYPE_STD` (the default), `LOG_OUTPUT_TYPE_FILE` or `LOG_OUTPUT_TYPE_BOTH`. The file is in `cerver_log_set_path ()` (`LOG_DEFAULT_PATH`, `/var/log/cerver`) and its thread syncs it to disk every `cerver_log_set_update_interval ()` seconds.
- **Quiet:** `cerver_log_set_quiet (true)` only prints success, warning and error messages.
- **Time:** none, time, date, or both, in local time or UTC.
- Every function takes a format like `printf ()`, checked by the compiler; a message longer than `LOG_MESSAGE_SIZE` is cut.

## Usage

``` c
cerver_log_set_time_config (LOG_TIME_TYPE_BOTH);
cerver_log_set_local_time (true);

cerver_log_success ("Cerver %s started", cerver->info->name);
cerver_log (LOG_TYPE_ERROR, LOG_TYPE_CLIENT, "Client %lu sent a bad packet", (unsigned long) client->id);
```

## Reference

### Configuration

``` c
void cerver_log_set_output_type (const LogOutputType type);
LogOutputType cerver_log_get_output_type (void);
unsigned int cerver_log_set_path (const char *pathname);
void cerver_log_set_update_interval (const unsigned int interval);
void cerver_log_set_time_config (const LogTimeType type);
LogTimeType cerver_log_get_time_config (void);
void cerver_log_set_local_time (const bool value);
void cerver_log_set_quiet (const bool value);
const char *cerver_log_time_type_to_string (const LogTimeType type);
const char *cerver_log_time_type_description (const LogTimeType type);
```

Set and get the output, the directory of the file (0 on success, 1 on error), the interval of its sync, the time of the messages, local time, quiet, and the names of the time types. Set them before `cerver_init ()` starts the file.

### Messages

``` c
void cerver_log (const LogType first_type, const LogType second_type, const char *format, ...);
void cerver_log_with_date (const LogType first_type, const LogType second_type, const char *format, ...);
void cerver_log_both (const LogType first_type, const LogType second_type, const char *format, ...);
```

A message with its types; with the date and time; and to the terminal and the file, ignoring quiet. `LOG_TYPE_NONE` as the second type prints only the first.

``` c
void cerver_log_msg (const char *msg, ...);
void cerver_log_error (const char *msg, ...);
void cerver_log_warning (const char *msg, ...);
void cerver_log_success (const char *msg, ...);
void cerver_log_debug (const char *msg, ...);
void cerver_log_raw (const char *msg, ...);
void cerver_log_line_break (void);
```

A message without a type, an error, a warning, a success, a debug message, a message without a format of cerver (like `printf ()`), and a line break.

## Types

| Type | What it is |
|---|---|
| `LogType` | `LOG_TYPE_NONE`, `ERROR`, `WARNING`, `SUCCESS`, `DEBUG`, `TEST`, `CERVER`, `CLIENT`, `CONNECTION`, `HANDLER`, `ADMIN`, `EVENT`, `PACKET`, `REQ`, `FILE`, `HTTP`, `GAME`, `PLAYER` |
| `LogOutputType` | `LOG_OUTPUT_TYPE_NONE`, `STD`, `FILE`, `BOTH` |
| `LogTimeType` | `LOG_TIME_TYPE_NONE`, `TIME`, `DATE`, `BOTH` |

## Constants

| Constant | Value |
|---|---|
| `LOG_DEFAULT_PATH` | `"/var/log/cerver"` |
| `LOG_MESSAGE_SIZE` | 4096 |
| `LOG_FILENAME_SIZE` | 1024 |
| `LOG_DEFAULT_UPDATE_INTERVAL` | 1 |
| `LOG_COLOR_RED` … `LOG_COLOR_RESET` | The ANSI colors |
| `LOG_TYPE_MAP (XX)`, `LOG_TIME_TYPE_MAP (XX)` | The X-macros of the types |

## See also

- [[Building]]: the debug logs of a development build
