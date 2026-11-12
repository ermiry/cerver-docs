# Histograms

`#include <cerver/utils/histogram.h>`

A `Histogram` keeps the count, the total, the minimum, the maximum and the distribution of a value, like the microseconds of a handler or the bytes of a packet. Every field is updated with atomics, so many threads add to the same histogram without a lock. The handlers and the request types of a cerver keep histograms (see [[Handlers]]).

## Concepts

`buckets[i]` counts the values up to 2^i, from 2^0 to 2^`HISTOGRAM_MAX_EXPONENT` (2^20), and the last bucket counts the bigger ones. A histogram without values has a minimum of `HISTOGRAM_EMPTY_MIN`, that the getters report as 0.

## Usage

``` c
static Histogram durations = HISTOGRAM_INITIALIZER;

histogram_add (&durations, elapsed_microseconds);

Histogram copy;
histogram_copy (&copy, &durations);
histogram_print ("Durations", "us", &copy);
```

## Reference

``` c
void histogram_init (Histogram *histogram);
void histogram_add (Histogram *histogram, const u64 value);
void histogram_copy (Histogram *destination, const Histogram *source);
unsigned int histogram_bucket (const u64 value);
double histogram_mean (const Histogram *histogram);
u64 histogram_min (const Histogram *histogram);
void histogram_print (const char *name, const char *unit, const Histogram *histogram);
```

Set a histogram without values, add a value, copy one with atomic reads, the bucket of a value, the mean and the minimum (0 without values), and print it with the log, with the unit of its values.

## Types

| Type | What it is |
|---|---|
| `Histogram` | `count`, `total`, `min`, `max`, `buckets` |

## Constants

| Constant | Value |
|---|---|
| `HISTOGRAM_MAX_EXPONENT` | 20 |
| `HISTOGRAM_BUCKETS` | 22 |
| `HISTOGRAM_EMPTY_MIN` | `UINT64_MAX` |
| `HISTOGRAM_INITIALIZER` | A static histogram without values |

## See also

- [[Handlers]]: the stats of the handlers and the request types
