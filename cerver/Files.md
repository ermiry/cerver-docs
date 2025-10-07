# Files

`#include <cerver/files.h>`, and the files of `#include <cerver/client.h>`.

Cerver sends and receives files between a cerver and its clients, in both directions, and has the file helpers that it uses itself: directories, copies, moves, reads, extensions and image types. A **files cerver** (`CERVER_TYPE_FILES`) answers the file requests of its clients with its `FileCerver`; a client answers the ones of its cerver with its own paths.

## Concepts

### Sending a file

A file travels in two parts: first a `PACKET_TYPE_REQUEST` packet with a `FileHeader`, an object of the [[Serializer]] with the `filename` and the `len` of the file, and right after it the bytes of the file, sent with `sendfile ()` straight from the file to the socket. The receiver reads the header, and then `len` bytes, with the receive buffer of the connection.

### The requests

| Request | Who sends it | What the other side does |
|---|---|---|
| `REQUEST_PACKET_TYPE_GET_FILE` | A client, with `client_file_get ()` | The cerver looks for the file in its paths and sends it, or answers `CERVER_ERROR_FILE_NOT_FOUND` |
| `REQUEST_PACKET_TYPE_SEND_FILE` | A client, with `client_file_send ()`, or a cerver with `file_send ()` | The receiver saves the file in its uploads path, or passes it to its upload handler, and runs its upload callback |

### Safe names

A `FileHeader` that comes from the network is used only if its filename is terminated with a NUL, not empty, a single name without `/`, and not `.` or `..`, so a request can not read past the header or reach a file outside of the configured paths. The receiver saves the file with that name inside its uploads path; a custom upload handler can clean it further with `files_sanitize_filename ()`.

### A files cerver

`cerver_create (CERVER_TYPE_FILES, ...)` creates the `FileCerver` of the cerver, in `cerver->cerver_data`:

- `file_cerver_add_path ()` adds a directory where the requested files are looked for, up to `FILE_CERVER_MAX_PATHS` (32);
- `file_cerver_set_uploads_path ()` sets where the uploaded files are saved;
- `file_cerver_set_file_upload_handler ()` replaces how an upload is saved, and `file_cerver_set_file_upload_cb ()` runs after each one.

## Usage

The cerver:

``` c
Cerver *cerver = cerver_create (
	CERVER_TYPE_FILES, "files-cerver",
	7000, PROTOCOL_TCP, false, CERVER_DEFAULT_CONNECTION_QUEUE
);

cerver_set_handler_type (cerver, CERVER_HANDLER_TYPE_THREADS);

FileCerver *file_cerver = (FileCerver *) cerver->cerver_data;
(void) file_cerver_add_path (file_cerver, "/var/files");
file_cerver_set_uploads_path (file_cerver, "/var/uploads");
```

The client asks for a file, that it saves in its own uploads path, and sends another one:

``` c
client_files_set_uploads_path (client, "/tmp/downloads");

(void) client_file_get (client, connection, "report.pdf");
(void) client_file_send (client, connection, "/home/user/photo.png");
```

## Reference

### A files cerver

#### `file_cerver_add_path ()`

``` c
u8 file_cerver_add_path (FileCerver *file_cerver, const char *path);
```

Adds a directory where the requested files are looked for.

- **Returns** 0 on success, 1 on error, like more than `FILE_CERVER_MAX_PATHS` paths.

#### `file_cerver_set_uploads_path ()`

``` c
void file_cerver_set_uploads_path (FileCerver *file_cerver, const char *uploads_path);
```

Sets the directory where the files that the clients upload are saved.

#### `file_cerver_set_file_upload_handler ()`

``` c
void file_cerver_set_file_upload_handler (
	FileCerver *file_cerver,
	u8 (*file_upload_handler) (
		Cerver *cerver, Client *client, Connection *connection,
		FileHeader *file_header,
		const char *file_data, size_t file_data_len,
		char **saved_filename
	)
);
```

Replaces how an upload is saved. The handler gets the header, the bytes of the file that arrived with its packet (`file_data`), and has to receive the rest, `file_header->len - file_data_len` bytes, from the socket of the connection. It sets `saved_filename` to a string allocated with `malloc ()`, and returns 0 on success, 1 on error.

#### `file_cerver_set_file_upload_cb ()`

``` c
void file_cerver_set_file_upload_cb (
	FileCerver *file_cerver,
	void (*file_upload_cb) (
		Cerver *cerver, Client *client, Connection *connection,
		const char *saved_filename
	)
);
```

Sets a callback that runs after each upload that was saved.

#### `file_cerver_search_file ()`

``` c
String *file_cerver_search_file (FileCerver *file_cerver, const char *filename);
```

Looks for a file in the paths of the files cerver.

- **Returns** the path where it was found, a `String` that the caller deletes, or `NULL`.

#### `file_cerver_send_file ()`

``` c
ssize_t file_cerver_send_file (
	Cerver *cerver, Client *client, Connection *connection,
	const char *filename
);
```

Looks for a file in the paths and sends it to a connection, or sends a `CERVER_ERROR_FILE_NOT_FOUND` error.

- **Returns** the bytes sent, or -1 on error.

#### `file_cerver_stats_print ()`

``` c
void file_cerver_stats_print (FileCerver *file_cerver);
```

Prints the requests, the files and the bytes that the files cerver sent and received.

### The files of a client

#### `client_files_add_path ()`, `client_files_set_uploads_path ()`, `client_files_set_file_upload_handler ()`, `client_files_set_file_upload_cb ()`, `client_files_search_file ()`

``` c
u8 client_files_add_path (Client *client, const char *path);
void client_files_set_uploads_path (Client *client, const char *uploads_path);

void client_files_set_file_upload_handler (
	Client *client,
	u8 (*file_upload_handler) (
		Client *client, Connection *connection,
		FileHeader *file_header,
		const char *file_data, size_t file_data_len,
		char **saved_filename
	)
);

void client_files_set_file_upload_cb (
	Client *client,
	void (*file_upload_cb) (Client *client, Connection *connection, const char *saved_filename)
);

String *client_files_search_file (Client *client, const char *filename);
```

The same as the ones of a files cerver, for the requests that a cerver sends to the client and the files that the client receives.

#### `client_file_get ()`

``` c
u8 client_file_get (Client *client, Connection *connection, const char *filename);
```

Asks the cerver for a file; the client saves it in its uploads path when it arrives.

- **Returns** 0 if the request was sent, 1 on error.

#### `client_file_send ()`

``` c
u8 client_file_send (Client *client, Connection *connection, const char *filename);
```

Sends a file to the cerver.

- **Returns** 0 if the file was sent, 1 on error.

### Sending

#### `file_send ()`

``` c
ssize_t file_send (Cerver *cerver, Client *client, Connection *connection, const char *filename);
```

Opens a file and sends it to a connection: its `FileHeader` in a packet and then its bytes.

- **Returns** the bytes sent, or -1 on error.

#### `file_send_by_fd ()`

``` c
ssize_t file_send_by_fd (
	Cerver *cerver, Client *client, Connection *connection,
	int file_fd, const char *actual_filename, size_t filelen
);
```

The same with a file that is already open; `actual_filename` is the name in its header.

- **Returns** the bytes sent, or -1 on error.

#### `file_header_is_valid ()`, `file_header_get ()`

``` c
bool file_header_is_valid (const FileHeader *file_header);
bool file_header_get (const Packet *packet, FileHeader *file_header);
```

Check that a `FileHeader` has a safe name (see above), and decode and check the one of a packet.

- **Return** `true` for a valid header.

### Directories

#### `files_create_dir ()`

``` c
unsigned int files_create_dir (const char *dir_path, mode_t mode);
```

Creates a directory if it does not exist.

- **Returns** 0 on success, 1 on error.

#### `files_create_recursive_dir ()`

``` c
unsigned int files_create_recursive_dir (const char *dir_path, mode_t mode);
```

Creates every directory of a path. A path that is already a directory is a success, also when another thread creates it at the same time; an empty path, or one that does not fit in `DIRNAME_DEFAULT_SIZE`, is an error.

- **Returns** 0 on success, 1 on error.

#### `files_get_from_dir ()`

``` c
DoubleList *files_get_from_dir (const char *dir);
```

Returns a list of `String`s with the names of the files of a directory, or `NULL`. The list deletes its strings with `dlist_delete ()`.

### Copies, moves and removals

#### `files_copy ()`

``` c
unsigned int files_copy (const char *source, const char *destination);
```

Copies a regular file with its permissions, replacing the destination. A copy that fails is removed.

- **Returns** 0 on success, 1 on error.

#### `files_copy_recursive ()`

``` c
unsigned int files_copy_recursive (const char *source, const char *destination);
```

Copies a file, a symbolic link (without following it) or a directory with everything in it; the directories of the destination can exist.

- **Returns** 0 on success, 1 on error.

#### `file_move_to ()`

``` c
int file_move_to (const char *actual_path, const char *saved_path);
```

Moves a file or a directory to `saved_path`, or inside it if it is a directory that exists, with `rename ()`, and between file systems with a copy that is removed after it. The paths it builds fit in `FILES_MOVE_SIZE`.

- **Returns** 0 on success, 1 on error.

#### `files_remove_recursive ()`

``` c
unsigned int files_remove_recursive (const char *path);
```

Removes a file, a symbolic link (without following it) or a directory with everything in it.

- **Returns** 0 on success, 1 on error.

### Reading

#### `file_exists ()`

``` c
bool file_exists (const char *filename);
```

Returns `true` if the path exists.

#### `file_read ()`

``` c
char *file_read (const char *filename, size_t *file_size);
```

Reads a whole file into a buffer terminated with a NUL, that the caller releases with `free ()`, and sets `file_size`, if it is not `NULL`, to the bytes that were read.

- **Returns** the buffer, or `NULL` on error.

#### `file_n_read ()`

``` c
char *file_n_read (const char *filename, const size_t n_bytes, size_t *n_read);
```

Reads up to `n_bytes` of a file, the same way.

#### `file_get_lines ()`

``` c
DoubleList *file_get_lines (const char *filename, const size_t buffer_size);
```

Returns a list of `String`s with the lines of a file, read with a buffer of `buffer_size` bytes, or `NULL`.

#### `file_open_as_file ()`, `file_open_as_fd ()`

``` c
FILE *file_open_as_file (const char *filename, const char *modes, struct stat *filestatus);
int file_open_as_fd (const char *filename, struct stat *filestatus, int flags);
```

Open a file as a `FILE` or as a file descriptor, and fill its `stat` in `filestatus`.

- **Return** the file, or `NULL` / -1 on error.

### Names and extensions

#### `files_sanitize_filename ()`, `files_sanitize_complete_filename ()`

``` c
void files_sanitize_filename (char *filename);
void files_sanitize_complete_filename (char *filename);
```

Remove from a name, in place, every character that is not a letter, a number, `-`, `_` or `.`; the second one also keeps `/`.

#### `files_get_file_extension ()`, `files_get_file_extension_reference ()`

``` c
char *files_get_file_extension (const char *filename);
const char *files_get_file_extension_reference (const char *filename, unsigned int *ext_len);
```

The extension of a name, without the dot: a copy that the caller releases with `free ()`, or a pointer into the name with its length. `NULL` without an extension.

### Images

#### `files_image_get_type ()`, `files_image_get_type_from_file ()`, `files_image_get_type_by_extension ()`

``` c
ImageType files_image_get_type (const char *filename);
ImageType files_image_get_type_from_file (const void *file);
ImageType files_image_get_type_by_extension (const char *filename);
```

The `ImageType` of a file by its magic bytes (opening it, or from bytes that were read), or by its extension: `IMAGE_TYPE_PNG`, `JPEG`, `GIF`, `BMP`, or `IMAGE_TYPE_NONE`.

#### `files_image_is_jpeg ()`, `files_image_extension_is_jpeg ()`, `files_image_is_png ()`, `files_image_extension_is_png ()`

``` c
bool files_image_is_jpeg (const char *filename);
bool files_image_extension_is_jpeg (const char *filename);
bool files_image_is_png (const char *filename);
bool files_image_extension_is_png (const char *filename);
```

Check the magic bytes or the extension (`jpg`, `jpeg`, `png`) of a file.

#### `files_image_type_to_string ()`, `files_image_type_extension ()`

``` c
const char *files_image_type_to_string (const ImageType type);
const char *files_image_type_extension (const ImageType type);
```

The name of an image type and its extension.

## Types

| Type | What it is |
|---|---|
| `FileCerver` | The paths, the uploads path, the upload handler and callback, and the stats of a files cerver |
| `FileCerverStats` | The requests, files and bytes sent and received |
| `FileHeader` | The object that goes before a file: `filename`, `len` |
| `ImageType` | `IMAGE_TYPE_NONE`, `PNG`, `JPEG`, `GIF`, `BMP` |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `FILE_CERVER_MAX_PATHS` | 32 | The paths of a files cerver |
| `DIRNAME_DEFAULT_SIZE`, `FILENAME_DEFAULT_SIZE` | 1024 | The buffers of a directory and a file name, with the NUL |
| `FILES_MOVE_SIZE` | 2048 | The paths that a move, a copy or a removal builds |
| `FILE_HEADER_VERSION` | 1 | The version of the schema of `FileHeader` |

## See also

- [[Packets]]: `PACKET_TYPE_REQUEST`
- [[Errors]]: `GET_FILE`, `SEND_FILE`, `FILE_NOT_FOUND`
- [[Sockets]]: how the bytes of a file are sent with a deadline
