# Cerver Documentation

Ermiry's Cerver Framework official documentation, for **cerver 1.8.1**, the core line (1.x).

Cerver is a framework to build fast and reliable servers in C, also usable from C++. It is a single shared library, `libcerver.so`, that only depends on pthreads, with what a server needs out of the box: TCP and UDP sockets in IPv4 and IPv6, the packets of its own protocol, handlers and threads, authentication and sessions, admins, a load balancer, file transfers, game lobbies, and the collections and utilities that all of them are built with.

## Start here

| Page | What it has |
|---|---|
| [[Quickstart]] | Build cerver and run a first cerver with its client |
| [[Building]] | Dependencies, the `make` options, Docker images and installing |
| [[Concepts]] | Cervers, clients, connections, packets, handlers and events, and the rules of memory and threads |
| [[Lifecycle]] | From `cerver_init ()` to `cerver_end ()`: create, configure, start, shutdown, drain and teardown |
| [[Configuration]] | Every setter of a cerver and a connection, with its default |

## Reference

| Area | Pages |
|---|---|
| Cerver | [[Cerver]], [[Handlers]], [[Events]], [[Errors]], [[Auth]], [[Sessions]], [[Admin]], [[Balancer]], [[Files]] |
| Clients | [[Clients]], [[Connections]] |
| Network | [[Packets]], [[Serializer]], [[Receive]], [[Network]], [[Sockets]], and the wire format in [[Protocol]] |
| Game | [[Game]], [[Lobby]], [[Players]], [[Gametypes]], [[Scores]] |
| Protocols | [[TCP]], [[UDP]] |
| Threads | [[Threads]], [[Threadpool]], [[Jobs]], [[Workers]], [[Managers]], [[Semaphores]] |
| Collections | [[Lists]], [[Trees]], [[Hashtables]], [[Queues]], [[Pools]], [[Arenas]], [[Freelists]] |
| Types and utilities | [[Types]], [[Strings]], [[Logs]], [[Histograms]], [[Base64]], [[SHA256]], [[Utilities]] |
| System | [[System]], [[Timers]], [[Input]], [[Version]] |

## More

| Page | What it has |
|---|---|
| [[Examples]] | The examples of the repository and what each one shows |
| [[Testing]] | The unit, integration and fuzz tests, and the sanitizers |
| [[Performance]] | Allocations, recycling, caches, buffer sizes and the limits of each handler type |
| [[Migration]] | The changes from cerver 1.7 to 1.8 and 1.8.1 |
| [[Troubleshooting]] | Common problems and how to solve them |

## Versions

| Line | Branch | What it is |
|---|---|---|
| **1.x** | `master` (releases), `core` (development), `beta` | The core of cerver, documented in this wiki |
| **2.0** | `http` (development), `http-beta` | The core plus an HTTP server; its pages arrive with the release 2.0 |
| **2.1** | `nightly` (development), `nightly-beta` | Experimental: an event loop, real keep alive and WebSocket |

Every header of the core is in `include/cerver/` and every page names the headers of the functions that it describes, like `#include <cerver/cerver.h>`. The functions marked `CERVER_PRIVATE` in the headers are internal and are not part of this reference.
