# Timers

`#include <cerver/timer.h>`

Clocks, elapsed times, sleeps and the texts of dates and times.

## Concepts

- `timer_get_timespec ()` reads `CLOCK_MONOTONIC_RAW`, that never jumps, for measuring; `timer_get_current_time ()` reads the time of the day, in seconds with microseconds.
- **`timer_get_gmt_time ()` and `timer_get_local_time ()` return the static buffer of `gmtime ()` and `localtime ()`**: they are not thread safe, and the next call overwrites the result. Copy it right away, or use `gmtime_r ()` in code that runs in several threads.

## Usage

``` c
TimeSpec *start = timer_get_timespec ();
// the work
TimeSpec *end = timer_get_timespec ();

cerver_log_msg ("took %f seconds", timer_elapsed_time (start, end));

timespec_delete (start);
timespec_delete (end);

String *now = timer_date_and_time_to_string (timer_get_local_time ());
str_delete (now);
```

## Reference

``` c
TimeSpec *timer_get_timespec (void);
void timespec_delete (void *timespec_ptr);
double timer_elapsed_time (TimeSpec *start, TimeSpec *end);
void timer_sleep_for_seconds (double seconds);
double timer_get_current_time (void);
```

A new reading of the monotonic clock and its release, the seconds between two readings, a sleep of fractional seconds, and the time of the day in seconds.

``` c
struct tm *timer_get_gmt_time (void);
struct tm *timer_get_local_time (void);
```

The current time in UTC and in local time (see the warning above).

``` c
void timer_time_to_string_actual (const struct tm *timeinfo, char *buffer);
String *timer_time_to_string (const struct tm *timeinfo);
void timer_date_to_string_actual (const struct tm *timeinfo, char *buffer);
String *timer_date_to_string (const struct tm *timeinfo);
void timer_date_and_time_to_string_actual (const struct tm *timeinfo, char *buffer);
String *timer_date_and_time_to_string (const struct tm *timeinfo);
String *timer_time_to_string_custom (const struct tm *timeinfo, const char *format);
```

The time as `24h`, the date as `day/month/year`, both, or with a format of `strftime ()`, in a buffer of `CERVER_TIMER_BUFFER_SIZE` bytes or in a new [[String|Strings]].

## Types

| Type | What it is |
|---|---|
| `TimeSpec` | `struct timespec` |

## Constants

| Constant | Value |
|---|---|
| `CERVER_TIMER_BUFFER_SIZE` | 128 |

## See also

- [[Logs]]: the dates of the messages
