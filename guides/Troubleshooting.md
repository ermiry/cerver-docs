# Troubleshooting

Common problems and how to solve them.

## Building

**`cannot find -lcerver` in `make examples`.** The examples link `bin/libcerver.so`: run `make` first.

**Errors after changing `TYPE`, `CC` or `SANITIZER`.** The objects of another build are reused: run `make clean` before building with other options.

**A `%ld` with a `u64` does not build.** The functions with a format are checked by the compiler; with `WERROR=1` the warning is an error. Use `%lu` with a cast, or `PRIu64`.

**It does not build on macOS.** Cerver builds in Linux, natively or in Docker (see [[Building]]); the macOS port is planned for cerver 1.9.

## Starting

**`handler type has NOT been configured`.** A TCP cerver needs `cerver_set_handler_type ()` with `CERVER_HANDLER_TYPE_POLL` or `CERVER_HANDLER_TYPE_THREADS`.

**`Failed to bind cerver socket`.** The port is in use, or it was just released: stop the other process, or use `cerver_set_reusable_address_flags (cerver, true)` to bind right after a restart. Ports below 1024 need privileges.

**A UDP cerver with admins fails to start.** A UDP cerver can not have admins (see [[UDP]]).

## Connecting

**A client of 1.7 (or of an earlier beta) can not talk with a cerver of 1.8.** The protocol changed: update both sides (see [[Migration]]).

**The client connects but never authenticates.** It needs its auth data before it connects (`connection_set_auth_data ()`); without it the connection stays on hold until the on hold timeout drops it (see [[Auth]]).

**A client keeps trying and fails after a minute.** The connection retries during its max sleep (60 seconds by default) with a backoff; lower it with `connection_set_max_sleep ()`, or check that the address and the family are right. A host name resolves to IPv4 and IPv6 addresses: an IPv4 only cerver refuses the IPv6 ones, and the client tries the next one.

**Connections are refused under load.** With threads, the thpool and its queue are full and the policy is `CERVER_OVERLOAD_POLICY_REJECT`, or with poll the array reached its max (`cerver_set_poll_fds ()`). See `total_rejected_connections` in the stats.

## Running

**The process dies with `SIGPIPE`.** A send to a connection that the client reset raised it. `cerver_start ()` ignores `SIGPIPE` by default; check that the process did not set a handler that kills it, or that the sends are not done outside of cerver without `MSG_NOSIGNAL`.

**Every connection waits when one client is slow.** With `CERVER_HANDLER_TYPE_POLL`, a slow direct handler, or a client that does not read, blocks the only thread. Give the slow handlers a queue, or use threads (see [[Performance]]).

**A crash in a handler with a queue.** A handler with a queue gets a `HandlerData *`, not the `Packet *`: the packet is `handler_data->packet` (see [[Handlers]]).

**A handler can not keep the packet.** The cerver deletes it after the handler returns; copy what you need, or take the packet with `cerver_set_app_handler_delete (cerver, false)` and delete it yourself.

**A use after free that only happens sometimes.** Connections, clients and packets are recycled: a pointer to one of them is not valid after it was deleted, even if it still looks right. Build with `SANITIZER=address`, that turns the recycling off.

## Stopping

**The teardown takes long.** It drains the work in progress for up to the drain timeout (32 seconds) plus a grace; a handler that never returns blocks it. Lower it with `cerver_set_drain_timeout ()`, or make the handlers return when the cerver stops.

**`teardown ended without releasing its memory`.** Some connections were still working after the drain, so their memory is kept to avoid a use after free; exit the process after it.

**`client_teardown ()` returns 1 from a handler.** A client can not be torn down from the thread of one of its connections: use `client_teardown_async ()`.

## See also

- [[Lifecycle]], [[Cerver]], [[Connections]]
