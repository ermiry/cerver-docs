# System

`#include <cerver/system.h>`

The resources of the host and of the process: file systems, memory, load, cpu, host name and the process itself, read from `statvfs ()`, `sysinfo ()` and `/proc`. They are for monitoring, like an update of a cerver or the requests of an [[Admin]]. They read Linux interfaces.

## Usage

``` c
SystemMemory memory;
if (system_get_memory (&memory)) system_print_memory (&memory);

SystemCpuSample first, second;
(void) system_cpu_sample (&first);
sleep (1);
(void) system_cpu_sample (&second);
cerver_log_msg ("cpu: %.1f %%", system_cpu_usage (&first, &second));
```

## Reference

### File systems

``` c
bool system_get_available_space (const char *path, unsigned long *available_space);
bool system_get_used_space (const char *path, unsigned long *used_space);
bool system_get_stats (const char *path, FileSystemStats *stats);
void system_print_stats (const FileSystemStats *stats);
```

The available and the used bytes of the file system of a path, its stats, and their print. They return `true` on success.

### Memory, load and cpu

``` c
bool system_get_memory (SystemMemory *memory);
void system_print_memory (const SystemMemory *memory);

bool system_get_load (SystemLoad *load);
void system_print_load (const SystemLoad *load);

bool system_cpu_sample (SystemCpuSample *sample);
double system_cpu_usage (const SystemCpuSample *first, const SystemCpuSample *second);
```

The memory of the host (`/proc/meminfo`), its load (`sysinfo ()`, with the online cpus), and a sample of the cpu times (`/proc/stat`). The usage between two samples is the percentage, 0 to 100, of busy time; 0 when the samples are equal or the second one is older.

### Host and process

``` c
bool system_get_hostname (char *hostname, const size_t size);

bool system_get_process (SystemProcess *process);
void system_print_process (const SystemProcess *process);
```

The name of the host, in a buffer of `size` bytes with the NUL (`false`, with an empty buffer, if it does not fit), and the resident memory, the threads and the open fds of this process (`/proc/self`).

## Types

| Type | What it is |
|---|---|
| `FileSystemStats` | `total`, `available`, `used`, `used_percentage` |
| `SystemMemory` | `total`, `free`, `available` (usable without swapping), `swap_total`, `swap_free`, in bytes |
| `SystemLoad` | `loads` (1, 5 and 15 minutes), `uptime`, `processes`, `cpus` |
| `SystemCpuSample` | `busy`, `total` |
| `SystemProcess` | `resident` bytes, `threads`, `open_files` |

## Constants

| Constant | Value |
|---|---|
| `MB`, `GB` | 2^20, 2^30 |
| `SYSTEM_LOADS` | 3 |
| `SYSTEM_HOSTNAME_SIZE` | 256 |

## See also

- [[Timers]], [[Input]], [[Version]]
