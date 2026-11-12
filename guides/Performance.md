# Performance

Where the time and the memory of a cerver go, what cerver does to keep them low, and the knobs that change them.

## Allocations

A connection that connects, sends packets and closes allocates very little, because the hot objects are **recycled** instead of released:

| Object | How |
|---|---|
| `Connection`, `Socket`, `Client`, `CerverReceive` | [[Freelists]] of up to 1024 of each, shared by every thread |
| `Packet` and its data | A cache of 16 in each thread for each class of data, from 2^6 to 2^12 bytes, plus shared lists |
| The stats, locks and conditions of a connection and a client | Embedded in them, not allocated apart |
| The jobs of a queue | The [[pool|Pools]] of the queue |

`test/allocations.sh` measures it with valgrind and fails above its limits: **at most 4 allocations per connection with `CERVER_HANDLER_TYPE_POLL`, 5 with `CERVER_HANDLER_TYPE_THREADS`, and 0 per packet of a handler with a queue**. A build with a sanitizer turns the recycling off, so it can see every use after free.

## Handler types

| | `CERVER_HANDLER_TYPE_POLL` | `CERVER_HANDLER_TYPE_THREADS` |
|---|---|---|
| Threads | One | One per connection being received |
| Memory per connection | An entry of the poll array | A thread and its stack |
| A slow handler | Delays every connection, unless it has a queue | Only delays its connection |
| A slow client | Its blocking sends delay every connection, up to the send timeout | Only delays its thread |
| Best for | Many connections, short work, handlers with queues | Fewer connections, work in the handlers |

The event loop of the 2.1 line removes these trade offs; in the 1.x, choose by the work of the handlers.

## Buffers and sizes

| Setting | Default | Effect |
|---|---|---|
| `cerver_set_receive_buffer_size ()` | 4096 | Bytes per `recv ()`: bigger buffers need fewer system calls for big packets, and more memory per connection |
| `cerver_set_max_received_packet_size ()` | 65507 | The biggest packet that is assembled |
| `cerver_set_thpool_n_threads ()` | 4 | Connections received at the same time with threads |
| `cerver_set_thpool_queue_size ()` | threads × 2 | Accepted connections that wait for a thread |
| `cerver_set_poll_fds ()` | 128 | The first size of the poll array, that doubles |
| `connection_set_receive_buffer_size ()` | 4096 | The same, for a client |

## Measuring

- **The stats** of a cerver (`cerver_stats_print ()`), of its handlers (`handler_get_stats ()`) and of its request types (`cerver_get_request_stats ()`) have histograms of the time of each handler and of the size of each packet.
- **The thpool** counts the jobs that waited for space and the times of the wait and the run (`thpool_get_stats ()`).
- **The benchmarks** (`make bench`) measure base64 and the handlers.

## See also

- [[Cerver]]: the handler types and the thpool
- [[Testing]]: the allocations and stress suites
