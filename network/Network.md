# Network

`#include <cerver/network.h>`

The addresses of cerver, in IPv4 and IPv6: resolving host names, parsing literal addresses with their zones, printing and comparing them, and the options of a socket. Cervers and clients use them for every bind and connect; an application uses them to work with the addresses of its connections.

## Concepts

### Families

- **A cerver** binds the family of its bind address (`cerver_set_bind_address ()`, see [[Cerver]]): `"0.0.0.0"` or `"127.0.0.1"` for IPv4, `"::"` or `"::1"` for IPv6. An IPv6 cerver with `cerver_set_ipv6_only (cerver, false)`, the default, also accepts IPv4 connections in the same socket (dual stack), that arrive as `::ffff:a.b.c.d`.
- **A client** resolves the host of its connection to every address of both families, alternating them and starting with the preferred one, and tries them in that order (see [[Connections]]). Each family keeps the order of `getaddrinfo ()` (RFC 6724).

### IPv4 inside IPv6

An address like `::ffff:192.168.1.2` is an IPv4 in an IPv6 socket. Cerver prints it as the IPv4 (`192.168.1.2`) and compares it as equal to the IPv4, so a client is the same no matter which socket it came from.

### Zones

A link local IPv6 address (`fe80::/10`) needs its interface: its **zone**, written after a `%`, as a name (`fe80::1%eth0`) or as an index (`fe80::1%2`). Cerver parses and prints the zones, and only takes them in link local addresses. `NETWORK_ADDRESS_STRING_SIZE` fits the longest address with its zone.

## Usage

``` c
// every address of a host, IPv6 first
NetworkAddresses addresses;
if (!network_resolve_addresses ("localhost", 7000, true, PROTOCOL_TCP, &addresses)) {
	char ip[NETWORK_ADDRESS_STRING_SIZE] = { 0 };
	for (unsigned int i = 0; i < addresses.count; i++) {
		if (!sock_ip_to_string_actual ((const struct sockaddr *) &addresses.addresses[i], ip)) {
			cerver_log_msg ("%s:%u", ip, sock_ip_port ((const struct sockaddr *) &addresses.addresses[i]));
		}
	}
}

// a literal address with its zone
struct sockaddr_storage address;
if (!network_parse_address ("fe80::1%eth0", 7000, &address)) {
	// bind or connect with sock_address_size (&address)
}
```

## Reference

### Addresses

#### `network_resolve_addresses ()`

``` c
unsigned int network_resolve_addresses (
	const char *host, const in_port_t port,
	const bool prefer_ipv6, const int protocol,
	NetworkAddresses *addresses
);
```

Resolves a literal ip, with its zone, or a host name to every address of the protocol (`PROTOCOL_TCP` or `PROTOCOL_UDP`), up to `NETWORK_MAX_RESOLVED_ADDRESSES` (16), alternating the families and starting with the preferred one.

- **Returns** 0 on success, 1 if the host could not be resolved.

#### `network_resolve_address ()`

``` c
unsigned int network_resolve_address (
	const char *host, const in_port_t port, const bool prefer_ipv6,
	struct sockaddr_storage *address
);
```

Resolves a host to a single address: one of the preferred family, or any other one if it has none.

- **Returns** 0 on success, 1 if the host could not be resolved.

#### `network_parse_address ()`

``` c
unsigned int network_parse_address (const char *ip, const in_port_t port, struct sockaddr_storage *address);
```

Parses a literal IPv4 or IPv6 address, without a port, into an address with the port. A link local IPv6 address can have its zone; a zone in any other address is not valid.

- **Returns** 0 on success, 1 if it is not a valid ip; a host name is not valid.

#### `network_hostname_to_ip ()`

``` c
char *network_hostname_to_ip (const char *hostname);
```

Resolves a host name to an ip, IPv4 or IPv6, with `getaddrinfo ()`, so it is thread safe.

- **Returns** the ip, allocated with `malloc ()`, or `NULL` if it could not be resolved.

#### `sock_ip_to_string_actual ()`, `sock_ip_to_string ()`

``` c
unsigned int sock_ip_to_string_actual (const struct sockaddr *address, char *ip_buffer);
char *sock_ip_to_string (const struct sockaddr *address);
```

Write the ip of an address, with its zone, in a buffer of at least `NETWORK_ADDRESS_STRING_SIZE` bytes (0 on success, 1 on error), or in a new string that the caller releases with `free ()`. An IPv4 inside an IPv6 address is written as the IPv4.

#### `sock_ip_port ()`

``` c
in_port_t sock_ip_port (const struct sockaddr *address);
```

Returns the port of an address, in host byte order.

#### `sock_ip_equal ()`

``` c
bool sock_ip_equal (const struct sockaddr *a, const struct sockaddr *b);
```

Returns `true` if both addresses have the same ip, port and zone; an IPv4 inside an IPv6 address is equal to the IPv4, and the flow info is not compared.

#### `sock_ip_is_mapped ()`

``` c
bool sock_ip_is_mapped (const struct sockaddr *address);
```

Returns `true` for an IPv4 inside an IPv6 address.

#### `sock_address_size ()`

``` c
socklen_t sock_address_size (const struct sockaddr_storage *address);
```

Returns the size of the address of its family, the one that `bind ()` and `connect ()` expect.

### Socket options

#### `sock_set_blocking ()`

``` c
bool sock_set_blocking (int32_t fd, bool blocking);
```

Clears or sets `O_NONBLOCK`: `false` makes the socket non blocking.

- **Returns** `true` on success.

#### `sock_set_timeout ()`, `sock_set_send_timeout ()`

``` c
int sock_set_timeout (int sock_fd, time_t timeout);
int sock_set_send_timeout (int sock_fd, time_t timeout);
```

Set `SO_RCVTIMEO` or `SO_SNDTIMEO` in seconds: a receive or a send that can not make progress in that time fails with `EAGAIN`.

- **Return** 0 on success, 1 on error.

#### `sock_set_reusable ()`, `sock_set_reusable_port ()`

``` c
int sock_set_reusable (int sock_fd);
int sock_set_reusable_port (int sock_fd);
```

Set `SO_REUSEADDR`, to bind a port right after another socket closed it, or `SO_REUSEPORT`, so several sockets bind the same port and the kernel balances them.

- **Return** 0 on success, 1 on error.

#### `sock_set_ipv6_only ()`

``` c
int sock_set_ipv6_only (int sock_fd, bool ipv6_only);
```

Sets `IPV6_V6ONLY` in an IPv6 socket; `false` also accepts IPv4 connections.

- **Returns** 0 on success, 1 on error.

## Types

| Type | What it is |
|---|---|
| `Protocol` | `PROTOCOL_TCP`, `PROTOCOL_UDP` |
| `NetworkAddresses` | The `addresses` that a host resolved to, and their `count` |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `NETWORK_ADDRESS_STRING_SIZE` | `INET6_ADDRSTRLEN + IF_NAMESIZE` | An ip with its zone |
| `NETWORK_MAX_RESOLVED_ADDRESSES` | 16 | The addresses of a host that are kept |
| `NETWORK_ZONE_SEPARATOR` | `'%'` | Between an IPv6 address and its zone |
| `NETWORK_IPV4_MAPPED_OFFSET` | 12 | Where the IPv4 of a mapped address starts |
| `MAX_UDP_PACKET_SIZE` | 65507 | The payload of an IPv4 UDP datagram |
| `MAX_PORT_NUM` | 65535 | |
| `IP_TO_STR_LEN`, `IPV6_TO_STR_LEN` | 16, 46 | The texts of an IPv4 and an IPv6 address |
| `SOCK_SIZEOF_MEMBER (type, member)` | | The size of a member of a struct |

## See also

- [[Sockets]]: the sockets of the connections and their sends
- [[Cerver]]: the bind address and the dual stack
- [[Connections]]: how a client tries the addresses
