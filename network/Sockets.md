# Sockets

`#include <cerver/socket.h>`

A `Socket` is the file descriptor of a connection with the locks that make it safe to use from many threads, and the sends with a **deadline** that every packet and file of cerver go through.

## Concepts

### Locks and references

| Member | What it protects |
|---|---|
| `read_mutex` | The receives, so a single thread reads a socket at a time |
| `write_mutex` | The sends, so the packets of different threads never mix |
| `close_mutex` | The close of the fd against a shutdown from another thread, so a shutdown never reaches an fd that was closed and given to another connection |
| `references` | The lifetime: the connection holds one, and each receive another one, so a thread can still read a socket after a handler deleted its connection |

A socket is deleted with its last reference (`socket_delete ()`), and it is never closed with the write mutex taken: `socket_close ()` shuts it down first, then waits for a send of another thread to end, and closes it.

### Sends with a deadline

A send can not wait forever for a peer that does not read. Every send has a **deadline**, a point in `CLOCK_MONOTONIC`, that several sends can share, so a single deadline limits a whole response. The sends use `MSG_NOSIGNAL` (a reset connection returns `EPIPE` instead of raising `SIGPIPE`) and `MSG_DONTWAIT`, and wait with `poll ()` when the socket is full, so the deadline is respected in blocking and non blocking sockets alike:

- an interrupted send (`EINTR`) is always retried;
- a full socket (`EAGAIN`) is retried while it can be written before the deadline;
- a file is sent with `sendfile ()`, in the space that the send buffer has, waiting with `poll ()` between the pieces.

The caller of a send of this page holds the write mutex of the socket. The sends of [[Packets]] already do.

## Usage

``` c
Socket *socket = connection->socket;

struct timespec deadline;
socket_deadline_create_milliseconds (&deadline, 2000);

(void) pthread_mutex_lock (socket->write_mutex);

int error = 0;
size_t sent = 0;
u8 failed = socket_send_buffer_until (socket, header, header_size, 0, &deadline, &sent, &error);
if (!failed) {
	failed = socket_send_file_until (socket, file_fd, NULL, file_size, &deadline, &sent, &error);
}

(void) pthread_mutex_unlock (socket->write_mutex);
```

## Reference

### Lifetime

#### `socket_create ()`, `socket_create_empty ()`, `socket_new ()`

``` c
Socket *socket_create (int fd);
void *socket_create_empty (void);
Socket *socket_new (void);
```

Create a socket for an fd, or without one (-1), with its locks and a reference. Sockets are recycled, like connections.

#### `socket_reference ()`, `socket_delete ()`

``` c
Socket *socket_reference (Socket *socket);
void socket_delete (void *socket_ptr);
```

Take a new reference to a socket (it returns the same socket), and release one; the socket is deleted with the last one.

#### `socket_close ()`

``` c
int socket_close (Socket *socket);
```

Shuts the fd down, waits for a send of another thread, closes it and sets it to -1, with the close mutex. The caller must not hold the write mutex.

- **Returns** the result of `close ()`, 0 if the socket was already closed.

#### `socket_replace ()`

``` c
void socket_replace (Socket *socket, const int fd);
```

Closes the fd of the socket, if it has one, and sets a new one, with the close mutex, like for a reconnection.

#### `socket_shutdown ()`

``` c
u8 socket_shutdown (Socket *socket);
```

Shuts down both ways of the fd, if it is still open, with the close mutex. It wakes up a thread that is blocked in a `recv ()` of the socket; the fd is only closed later, so it can not be reused while that thread uses it.

- **Returns** 0 on success, 1 if the socket is closed or the shutdown failed.

### Deadlines

#### `socket_deadline_create ()`, `socket_deadline_create_milliseconds ()`

``` c
void socket_deadline_create (struct timespec *deadline, const time_t timeout);
void socket_deadline_create_milliseconds (struct timespec *deadline, const u32 timeout);
```

Set a deadline `timeout` seconds or milliseconds from now; 0 uses `CERVER_DEFAULT_SEND_TIMEOUT`.

#### `socket_deadline_remaining ()`

``` c
int socket_deadline_remaining (const struct timespec *deadline);
```

Returns the milliseconds left until the deadline, 0 if it has expired.

#### `socket_wait_writable ()`

``` c
bool socket_wait_writable (const int sock_fd, const struct timespec *deadline);
```

Waits with `poll ()` until a full socket can be written again.

- **Returns** `true` if it can be written before the deadline.

#### `socket_send_can_retry ()`

``` c
bool socket_send_can_retry (const int sock_fd, const int error, const struct timespec *deadline);
```

Decides if a send that failed with `error` is retried: always after `EINTR`, after `EAGAIN` while the socket can be written before the deadline.

### Sends

All of them set `sent`, if it is not `NULL`, to the bytes that were sent, and `error`, if it is not `NULL`, to the `errno` of a failure, `EAGAIN` when the deadline expired. They return 0 on success, 1 on error.

#### `socket_send_vector_until ()`, `socket_send_vector ()`

``` c
u8 socket_send_vector_until (
	const Socket *socket,
	struct iovec *parts, int n_parts,
	int flags, const struct timespec *deadline,
	size_t *sent, int *error
);

u8 socket_send_vector (
	const Socket *socket,
	struct iovec *parts, int n_parts,
	int flags, const time_t timeout,
	size_t *sent, int *error
);
```

Send every part with as few system calls as the socket takes, like `writev ()`. The parts are changed to skip what was sent. The second one uses a deadline of `timeout` seconds from now.

#### `socket_send_buffer_until ()`, `socket_send_buffer ()`

``` c
u8 socket_send_buffer_until (
	const Socket *socket, const void *buffer, const size_t size,
	int flags, const struct timespec *deadline,
	size_t *sent, int *error
);

u8 socket_send_buffer (
	const Socket *socket, const void *buffer, const size_t size,
	int flags, const time_t timeout,
	size_t *sent, int *error
);
```

The same with a single buffer.

#### `socket_send_file_until ()`, `socket_send_file ()`

``` c
u8 socket_send_file_until (
	const Socket *socket,
	const int file_fd, off_t *offset, const size_t size,
	const struct timespec *deadline,
	size_t *sent, int *error
);

u8 socket_send_file (
	const Socket *socket,
	const int file_fd, const size_t size,
	const time_t timeout,
	size_t *sent, int *error
);
```

Send `size` bytes of a file with `sendfile ()`, from `offset`, that moves, or from the current offset of the file when it is `NULL`, until every byte was sent or the deadline expires. A file that ends before `size` bytes fails with an `error` of 0.

## Types

| Type | What it is |
|---|---|
| `Socket` | `sock_fd`, its buffer, its read, write and close mutexes, and its references |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `SOCKET_SEND_FILE_MIN_ROOM` | 4096 | The least bytes of each piece of a file to a blocking socket |
| `SOCKET_SEND_FILE_ROOM_DIVISOR` | 2 | The part of the free space of the send buffer that each piece takes |
| `SOCKET_MILLISECONDS_PER_SECOND`, `SOCKET_MICROSECONDS_PER_MILLISECOND`, `SOCKET_NANOSECONDS_PER_MILLISECOND`, `SOCKET_NANOSECONDS_PER_SECOND` | | Units of the deadlines |

## See also

- [[Network]]: the addresses and the options of a socket
- [[Packets]]: the sends of packets
- [[Files]]: the sends of files
