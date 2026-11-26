# rservd - infdev

**A single-file HTTP/1.1 server, written in C, designed to be small,
portable, and honest about what it does.**

<h3>

*Document author: DeepSeek*

*Document editor: Hazardous*

*Program development team:*

*Hazardous - Lead programmer*

*Deepseek - Tester*

*Version: Thu Nov 12 18:54:07 2026*

*License: BSD-3-Clause*</h3>

---

## What it is

`rservd` (retro serve daemon) is a static-content HTTP/1.1 server in
one `.c` file, roughly 30 KB of compiled binary, that runs on FreeBSD,
Linux, Termux, and any other POSIX-compliant system with a C99 compiler
and pthreads. It serves files from a document root, generates directory
listings on the fly, provides three built-in diagnostic endpoints under
`/about:`, and does not accept uploads of any kind.

It has no external dependencies beyond libc and libpthread. There is no
config file parser, no module loader, no plugin system, no scripting
engine. Every decision about what to serve and how to serve it is either
compiled into the binary or passed on the command line.

The design target was not "Apache replacement" or "Nginx competitor". It
was "a server small enough to fit on a floppy disk, honest enough to fit
in a single file, and portable enough to run on anything that has `cc`."

---

## Why this exists

The web has been built on servers that are large, opaque, and full of
features that most sites do not use. Apache ships with a config language
so large that whole books exist to explain it. Nginx is faster but brings
a module system and a DSL. Both of them are excellent — and both of them
are more than most people need.

`rservd` is what happens when you take the opposite approach: strip the
server down to what is actually necessary to serve static content, and
leave everything else out. There is no reason a static site needs a
configuration file. There is no reason a fileserver needs to speak
`POST`. There is no reason an HTTP server needs 3 MB of binary and 30 MB
of resident memory to hand over an HTML page.

The codebase follows the same principle. There is one `handle_client()`
function, one `main()`, one MIME table, one privilege-drop function. You
can read the whole source in an evening.

---

## Features

| Category | What it does |
|---|---|
| **Serving** | Static files from a document root, MIME type detection by extension via `bsearch` |
| **Directory listing** | Auto-generated `index.html` / `index.htm` lookup; falls back to a styled directory listing |
| **Diagnostics** | `/about:server`, `/about:test`, `/about:blank` — compiled-in dynamic endpoints |
| **HTTP** | `GET` and `HEAD` only; `Connection: close` on every response; no keep-alive |
| **Concurrency** | Thread-per-connection, bounded by `-t` (default 10) |
| **Performance** | `sendfile(2)` on FreeBSD (zero-copy), `read`/`write` fallback with 8 KB buffer elsewhere |
| **Privilege separation** | Binds as `root`, drops to `www` before accepting requests |
| **Path security** | Rejects any URI containing `..` or `%2e%2e`; validates every served file against the document root |
| **Error pages** | 403, 404 with stylized Holo-Red HTML |
| **Logging** | Per-request method + path to stdout; `-v` for verbose lifecycle messages |

---

## Building

`rservd` builds with any C99 compiler. The provided `Makefile` is written
in POSIX make syntax, which means it works with:

- `bmake` on FreeBSD, NetBSD, OpenBSD, DragonFly
- `gmake` (GNU Make) on Linux, macOS, Termux
- `make` on SunOS, Haiku, and most other Unix systems

### Quick build

```sh
make
```

This produces a `rservd` binary in the current directory.

### Build with a specific compiler or flags

The `Makefile` uses `?=` assignment (well, `=` in the current version,
but you can override from the command line):

```
make CC=clang CFLAGS="-O2 -march=native -Wall -Wextra -std=c99"
```

### Install to the system

```
sudo make install
```

By default this places the binary in `/usr/local/bin/rservd`. To install
elsewhere:

```
sudo make install PREFIX=/opt/rservd
```

Or stage for a package:

```
make install DESTDIR=/tmp/staging PREFIX=/usr/local
```

### Clean

```
make clean
```

### Makefile in full

```
.POSIX:

CC      = cc
CFLAGS  = -O2 -g -Wall -Wextra -std=c99 -D_POSIX_C_SOURCE=200809L
LDFLAGS = -pthread

PREFIX  = /usr/local
BINDIR  = $(PREFIX)/bin

TARGET  = rservd
SRC     = rservd.c

all: $(TARGET)

$(TARGET): $(SRC)
	$(CC) $(CFLAGS) $(SRC) -o $(TARGET) $(LDFLAGS)

install: $(TARGET)
	mkdir -p $(DESTDIR)$(BINDIR)
	install -m 755 $(TARGET) $(DESTDIR)$(BINDIR)

clean:
	rm -f $(TARGET)

.PHONY: all install clean
```

The `.POSIX:` directive at the top tells any make implementation to
behave according to POSIX rules, which is what makes the file portable.
`-pthread` (rather than `-lpthread`) is used because it is the portable
flag across GCC and Clang, and it activates pthreads on systems where
they are in libc (FreeBSD, macOS) as well as systems where they are in
a separate library (Linux).

---

## Running

### Basic usage

```
./rservd
```

With no arguments, the server binds to `0.0.0.0:8080`, serves from
`/etc/rserv/www` (creating it if needed), and drops privileges to `www`
if launched as root.

### Full usage

```
rservd [-p port] [-b backlog] [-t threads] [-u] [-v]

  -p port      Port to bind to. Default: 8080.
  -b backlog   Listen backlog length. Default: 10.
  -t threads   Maximum concurrent threads. Default: 10.
  -u           User mode. Serve from ./www instead of /etc/rserv/www.
               Does not attempt to drop privileges.
  -v           Verbose mode. Prints lifecycle messages to stdout.
```

### Examples

Serve from `/etc/rserv/www` on port 8080 as the `www` user:

```
sudo ./rservd
```

Serve from `./www` on port 2009, ten concurrent threads, verbose:

```
./rservd -p 2009 -t 10 -u -v
```

Listen on the standard HTTP port with a large backlog:

```
sudo ./rservd -p 80 -b 128 -t 32
```

### Where it looks for files

`rservd` uses one of two document roots, depending on whether `-u` was
passed on the command line:

| Mode ↕▾ | Document root ↕▾ | When to use ↕▾ |
|---|---|---|
| −**Admin** (no `-u`) | `/etc/rserv/www` | Production. Requires creating `/etc/rserv` and dropping to `www`. |
| **User** (`-u`) | `./www` (relative to cwd) | Development. No privilege changes, no filesystem writes outside `./www`. |

Deepseek: Both modes read at least: the document root itself, and whatever files
or subdirectories you place under it.

Hazard: Reads almost everything in document root excluding all
files containing *..* or *%2e%2e* pattern in their paths, or files under the name of *Built-in endpoints*

### Directory structure

After running in admin mode for the first time:

```
/etc/rserv/                     — auto-created by pathgen()
/etc/rserv/www/                 — auto-created by pathgen()
/etc/rserv/www/index.html       — you provide this
/etc/rserv/www/pages/           — you provide this, and so on
```

In user mode:

```
./www/                          — you provide this
./www/index.html                — you provide this
```

---

## How a request is handled

The lifecycle of a single request, in order:

1. **`accept(2)`** — the main thread pulls a connection off the listen
queue. If `active_threads >= max_threads`, the connection is answered
with `503 Service Unavailable` and closed. No thread is spawned.
2. **`pthread_create(3)`** — a new thread is spawned to handle this
connection. The connection's file descriptor is passed as the thread
argument, cast through `intptr_t` (which is the portable way to pass
an integer through `void *`).
3. **`read(2)`** — the first 4096 bytes of the request are read. No
attempt is made to read headers beyond what fits in one `read()`.
4. **Parse** — `sscanf("%7s %255s", method, path)` extracts the method
and the requested path. Anything that doesn't match this shape is
treated as a malformed request and closed.
5. **Route** — the path is compared against the three built-in
`/about:` endpoints. If it matches one, that endpoint is served.
6. **Security check** — if the path contains `..` or `%2e%2e`, the
request is rejected with a `403` immediately.
7. **Path construction** — the requested path is joined with the
document root. If the path ends in `/`, the server looks for
`index.html` then `index.htm` in that directory and serves the first
one found.
8. **`realpath(3)`** — the fully constructed path is canonicalized.
This resolves any remaining `..`, symlinks, and relative components.
9. **`open(2)` with `O_NOFOLLOW`** — the canonical path is opened for
reading, and symlinks are rejected at the syscall level.
10. **Prefix check** — the canonical path is verified to begin with the
document root. Paths that escape the document root after `realpath`
are rejected with `403`.
11. **`fstat(2)`** — the opened file's metadata is read. Directories
trigger the directory listing; regular files proceed; anything else
(device nodes, FIFOs, sockets) is rejected with `404`.
12. **Response** — `HTTP/1.1 200 OK` is written with the correct
`Content-Type` (from the MIME table) and `Content-Length` (from
`fstat`). On FreeBSD, `sendfile(2)` writes the file content directly
from the page cache into the socket. On other systems, an 8 KB
`read`/`write` loop does the same.
13. **`close(2)`** on both file and socket. The thread then decrements
`active_threads` under the mutex and exits.

Every path through the function converges on the `cleanup:` label at
the bottom, which is where `active_threads` is decremented. There is
exactly one place where the thread counter goes down, which is why it
never leaks.

---

## Built-in endpoints

These are compiled into the binary and take priority over any file that
might exist at the same path.

### `/about:server`

Returns a page describing the running system. Content is generated at
request time from `uname(2)` and from the `poweredlink` variable that
`main()` initializes based on `buffer.sysname`.

What appears on the page:

- Kernel name (from `uname(2).sysname`)
- A "powered by" image — FreeBSD's powerlogo or Wikipedia's Linux icon
- Architecture (from `uname(2).machine`)

The page also has a "Rservd" button that opens `kernelflare.c-om.net` in
a popup — this is the KernelFlare project page.

### `/about:test`

Minimal test page. Returns a single `<h1>Rservd test, btw</h1>`. Exists
so you can verify the server is alive without reading a real file.

### `/about:blank`

Returns a truly empty HTML document. Useful for browser tests where you
want a valid HTTP 200 response with no content of interest.

---

## Directory listing

If a directory is requested and neither `index.html` nor `index.htm` is
present, the server generates a directory listing on the fly.

The listing is a styled HTML page using the same Holo-Red theme as the
error pages. Each entry is a clickable link, styled as a "pane" — a
full-width block that lights up on hover. The `.` and `..` entries are
present, but they carry `rel="nofollow"` so that recursive crawlers
(including `wget -r` and `srcbox`) do not descend into a parent
directory and get stuck in a loop.

File links use the standard `<a href="...">` mechanism, not JavaScript
`onclick` — this is important because tools like `wget`, `curl`, `lynx`,
and search engine crawlers do not execute JavaScript and would fail to
find any files if the links were scripted.

---

## MIME types

MIME detection is by extension, using `strrchr` to find the last `.` in
the filename and a sorted table of 128+ entries looked up with `bsearch`.

The table covers: text, code, images, audio, video, archives, fonts,
virtual machine disk images, certificates, and several project-specific
formats (`.qcow`, `.vdi`, `.vhd`, `.vbox`, `.vbox-prev`).

Textual types (`text/*`, `application/json`, `application/xml`,
`application/javascript`, `text/yaml`) are served with
`; charset=utf-8` appended to the `Content-Type` header. Binary types
are not.

If no extension is found, or the extension is not in the table, the
server falls back to `application/octet-stream`, which tells the browser
to download rather than attempt to render.

The table must remain sorted. `bsearch` requires a sorted array, and if
you add an entry in the wrong place, that extension will silently fail
to match. Every entry is compared with `strcasecmp`, so `.PNG` and
`.png` are treated as equivalent.

---

## Security model

The full model is documented in `SECURITY.md`, but the short version:

- **`..` is rejected outright.**
- 
Deepseek: Not normalized, not resolved — rejected.
Any URI containing `..` or `%2e%2e` gets the same `403` response,
whether the resulting path would exist or not.

Hazard: This is the same
principle as a login screen saying "wrong username or password" —
it does not tell an attacker which part of the attempt was correct.

- **Errors do not leak information.** 403, 404, and (planned) 405 all
return fixed content that does not distinguish "file does not exist"
from "file exists but you can't see it".
- **Read-only by design.** The server has no upload path. There is no
code path that writes to the filesystem.
- **Privilege separation.**

Deepseek: In admin mode, the process binds to its
port as `root`, then drops to `www` before accepting any connection.
`drop_privileges()` calls `setuid(0)` afterwards to verify the drop
actually took — if `setuid(0)` succeeds, ~~the process aborts with a
critical error.~~


Hazard: Does not abort, instead shows the
```rserve_e: CRITICAL, got access to root user!```
error message

- **Three checks on every file.** `realpath` to canonicalize,
`strncmp` against the document root to reject escapes, and
`O_NOFOLLOW` + `fstat` to reject anything that is not a regular file.
- **Thread cap.** The `-t` flag bounds concurrent connections. Excess
connections get `503 Service Unavailable` and are closed without
spawning a thread.

The current `infdev` build does **not** yet filter non-`GET`/`HEAD`
methods explicitly — a `POST /index.html` will be handled as if it were
`GET /index.html`. A 405 branch is planned for the next revision.

---

## Known limitations

The following are open attack surfaces or missing features that anyone
deploying `rservd` should be aware of:

- **No HTTPS.** `rservd` speaks plain HTTP/1.1. Terminate TLS upstream
(a reverse proxy, `stunnel`, Tailscale Funnel, or similar) if you
need encryption.
- **No `Range` header.** Video and audio are served as whole files.
Browser seeking will not work until the full file is transferred.
- **No rate limiting.** Slowloris-style attacks are mitigated only by
the `-t` cap. A dedicated rate limiter would require per-IP state
tracking, which the current architecture does not implement.
- **No CGI.** Dynamic endpoints (`/about:*`) are compiled in. External
scripts are not run.
- **No keep-alive.** Every response closes the connection. This is a
trade-off — it eliminates one class of resource exhaustion at the
cost of per-request connection overhead.
- **No method filtering (yet).** As noted above, `POST` and other
methods are currently routed as if they were `GET`. This will be
fixed.
- **No signal handling.** `SIGINT` and `SIGTERM` terminate the process
immediately. A graceful-shutdown handler is not yet implemented.
- **Single-`read` request parsing.** Requests are parsed from the first
4096 bytes. Requests with large headers or bodies will not be fully
read, but since the server does not parse bodies, this is only a
compliance concern, not a security one.
- **`listen(2)` after privilege drop.** The current `infdev` build
calls `listen()` after `drop_privileges()`. This is fine on FreeBSD
and Linux (listen on an already-bound socket does not require
privileges), but a future revision may move `listen()` before the
drop for maximum portability.

---

## The name

`rservd` is short for "retro serve daemon". The "d" is the Unix
convention for "daemon". The word "retro" is intentional: the interface
is meant to look like a 1990s terminal, the MIME table is deliberately
small and hand-curated, and the error pages use Holo-Red styling
reminiscent of Android 4.x.

The visual design of the error pages and directory listing uses:

- Background: `#0f0f0f` to `#272727` gradient
- Accent: `#FF4444` (Holo-Red)
- Text: `#dbdbdb` / `#bbbbbb`
- Monospace fonts throughout
- A blinking `_` cursor in the header, styled with CSS `@keyframes`

~~Deepseek: The client-side font list references `'admin'` (Better VCR), which is
the intended display font. On systems where `admin` is not installed,
the browser falls back to `system-ui`.~~

Hazard: Hotfixed, now uses only monospace

---

## Compiling on other systems

`rservd` is written to be portable. The `sendfile()` call is guarded by
`#if defined(__FreeBSD__)` — on other systems, the fallback `read`/`write`
loop runs. The `pthread` API used is fully POSIX.

### Linux

```
cc -O2 -pthread -D_POSIX_C_SOURCE=200809L rservd.c -o rservd
```

Or, with the Makefile:

```
gmake
```

### Termux (Android)

```
pkg install clang make
gmake
./rservd -p 8080 -u
```

The `-u` flag is required on Termux, since Android does not allow
binding to privileged ports and does not have a `www` user.

### macOS

```
cc -O2 -pthread rservd.c -o rservd
```

The FreeBSD `sendfile` codepath is skipped; the fallback runs. `rservd`
has not been tested on macOS as of `infdev`, but it should compile.

### MINIX

Untested. The pthreads requirement is the main concern — MINIX's
threading support is minimal. This will be investigated during porting.

### SunOS / illumos

The `sendfile` call is different on SunOS (different signature, different
argument order). The `#if defined(__FreeBSD__)` guard skips it, and the
fallback `read`/`write` loop runs. The binary should compile; whether it
runs will depend on whether SunOS's pthread library is compatible with
the POSIX subset `rservd` uses.

---

## What comes next

The immediate roadmap:

1. **Method filtering** — reject `POST`, `PUT`, `DELETE`, and every
method other than `GET` and `HEAD` with a proper `405 Method Not
Allowed` response.
2. **More MIME types** — the table is currently at 128+ entries. The
next revision doubles it. The current list was deliberately kept
short to keep the binary small enough for a floppy; the second
round of additions will be the point at which the floppy target
starts to be a stretch.
3. **CGI** — the next major feature. `rservd` will gain the ability to
spawn external scripts with `fork()` + `execve()`, passing CGI
environment variables (`REQUEST_METHOD`, `QUERY_STRING`,
`PATH_INFO`, `SCRIPT_NAME`, `SERVER_PROTOCOL`, `HTTP_*` headers)
and reading their stdout into the response. This is where the
security surface grows — the current design is safe because it
does not execute anything.
4. **Range support** — `Range: bytes=N-M` parsing with `206 Partial
Content` responses, needed for video seeking.
5. **Signal handling** — graceful shutdown on `SIGINT`/`SIGTERM`.
6. **Man page** — the current README will be split; the man page will
cover flags and endpoints, the README will cover architecture and
porting.

Things that are deliberately **not** on the roadmap:

- HTTPS. `rservd` is not a TLS terminator. Use a reverse proxy.
- CGI as a general-purpose runtime. CGI is being added for a specific
reason — a handful of dynamic endpoints that need to touch the
filesystem — not as a general scripting engine.
- Module system. There is no plugin architecture, and there will not
be one. `rservd` is one binary.
- Configuration files. Everything is on the command line, and that is
intentional.
- Keep-alive. It adds per-connection state and complicates the
threading model. The current design does not need it.

---

## Why it matters

Every server you use today is the result of thousands of design
decisions made over decades by people who had reasons at the time. Most
of those reasons were good. Some were not. A few were bad enough that
the code carries the scars of them still, thirty years later.

`rservd` is not a replacement for any of those servers. It is not
faster, more secure, more stable, or more featureful. What it is, is
readable. Every line is there for a reason. If it seems like it might
not be, ask why, and the answer will usually be "because otherwise a
path would escape the document root" or "because otherwise a bot
would enter an infinite recursion" or "because otherwise the process
would still have root privileges when it shouldn't."

This is what it looks like when you strip a server down to the
essentials and check every assumption at the door.

---

## Contact

- **Project:** KernelFlare
- **Server source:**
Deepseek: `http://kernelflare.c-om.net` on the KernelFlare Tailscale
Hazard: here
- **Bug reports:** `kernelflare@gmail.com`
- **Project page:** `kernelflare.c-om.net`

The original of code was presented by Tony Banters (c-httpd.c)

The original of this documentation were written by
DeepSeek.

The editor, who is responsible for stylistic consistency with
the rest of the KernelFlare site, is Hazardous.

---

## License

bsd-3-clause

The MIME type table was compiled from the IANA media types registry and
from Apache's `mime.types`, both of which are public domain / freely
reusable for this purpose. No code was copied from Apache or from any
other server; the table is derived from public reference data.

---

*Built with `cc`. Shipped on a floppy. Served from a C file.*

