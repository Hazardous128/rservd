# Rservd — Security Model

Rservd is small. Small means small attack surface. But the model below
is deliberate, not accidental — every line of the code was written with
the assumption that the network is hostile.

## 🛡️ Path Traversal — Total Rejection

Any URI containing `..` or `%2e%2e` returns **403 immediately** — whether
the resulting path would exist or not.

This is intentional. The server refuses to say. The attacker gets the
same response for a real exploit attempt (`GET /../../../etc/passwd`) and
for an innocent typo (`GET /pages/../index.html`). Same principle as a
login screen that says *"wrong username or password"* no matter which of
the two was wrong — you never learn which half you got right.

Applied to filesystem paths: the server never tells you whether a
traversal target exists. That's information you don't need.

**Not a bug. Feature.** Documented as such.

## 🔇 No Information Leak in Error Pages

403, 404, and 405 responses return fixed content. The server never
distinguishes:
- "file exists but you can't access it" (403)
- "file doesn't exist at all" (404)
- "you used the wrong method" (405)

Every rejection looks identical from the outside. Combined with the
`..` policy above, an attacker scanning for files learns only
"something is wrong" — never *what*.

## 📖 Read-Only By Design

Only `GET` and `HEAD` are accepted. `POST`, `PUT`, `DELETE`, `TRACE`,
`OPTIONS`, `PATCH`, `CONNECT` — every other method falls into the
default case and gets rejected.

This means: **there is no upload path**. No forms, no file submission,
no CGI state changes. Rservd serves files. It doesn't accept them.

## 🧬 Privilege Separation

In admin mode (`-u` not set):

1. `socket()` — allocate the listener.
2. `bind()` — bind to the (possibly privileged) port **as root**.
3. `pathgen()` — create `/etc/rserv/www` if it doesn't exist.
4. `listen()` — start accepting.
5. `drop_privileges("www")` — **immediately after bind**.
6. `accept()` loop — every request handled as `www`.

If the server is compromised after step 5, the attacker inherits `www`
privileges, not root. On a typical FreeBSD box that means: no writes
outside `/home/www`, no ability to bind privileged ports, no ability to
spawn setuid-root helpers.

`drop_privileges()` also calls `setuid(0)` afterwards to verify the drop
actually took. If `setuid(0)` succeeds — the process aborts with a
critical error. This catches broken kernels or misconfigured containers.

## 📂 Filesystem Isolation — Three Checks

Before any file is opened and served:

1. **Path normalization** — `realpath(filename, resolved_path)` resolves
   the absolute path, resolving symlinks and `..` *after* the rejection
   pass above. This gives the canonical path.
2. **Prefix check** — `strncmp(resolved_path, root_dir, strlen(root_dir))`
   verifies the canonical path begins with the document root. This is
   the line that actually defeats symlink attacks, because a symlink
   inside `www/` that points outside will resolve to a path that does
   not start with `root_dir`.
3. **Type check** — `fstat(file, &info)` + `S_ISREG(info.st_mode)`.
   Directories, device nodes, FIFOs, sockets — all rejected. Only
   regular files are served.

Plus: `open(filename, O_RDONLY | O_NOFOLLOW)` — symlinks are rejected
at the syscall level as a belt-and-suspenders measure. If checks 1–3
were ever bypassed, `O_NOFOLLOW` still prevents the open.

## 🧵 Thread Bounds

`-t N` (default 10) caps concurrent request handlers. When `active_threads
>= max_threads`, the server answers with `HTTP/1.1 503 Service Unavailable`
and closes the connection. No new threads are spawned.

This is the DoS mitigation. An attacker opening 5000 sockets
simultaneously hits the thread cap, and the excess sockets get dropped
with 503 — not queued, not accepted, not spawned. The kernel accepts
them, the server rejects them.

## 🧊 HEAD is Honored

`HEAD` requests return headers only. Content-Length, Content-Type, and
status are correct — only the body is omitted. This is what `wget -r`,
`curl -I`, and search engine crawlers expect.

Rservd's `autoindex` uses `<a href="...">` for every file, with the
`rel="nofollow"` attribute on `.` and `..` entries. This tells
recursive crawlers to *see* the directory navigation links but not
follow them — so `wget -r` on a directory containing autoindex works
without entering infinite recursion.

## ⚠️ Known Limitations (Infdev)

The following are **not** implemented and should be treated as open
attack surfaces by anyone deploying rservd in production:

- **No HTTPS.** Rservd speaks plain HTTP/1.1. Terminate TLS upstream
  (Tailscale Funnel, HAProxy, stunnel) if you need encryption.
- **No Range header.** Video and audio files serve whole, not in
  segments. Seeking in the browser will stall until the full file is
  buffered.
- **No Rate Limiting.** Slowloris-style attacks (many connections, each
  sending one byte per minute) are mitigated only by `-t` cap and by
  TCP timeouts, not by the server itself.
- **No CGI.** Static files only. Dynamic endpoints (`/about:server`,
  `/about:test`, `/about:blank`) are compiled-in, not scripted.
- **No HTTP keep-alive.** Every response sends `Connection: close`. This
  is a DoS trade-off — it eliminates one class of resource exhaustion
  but adds per-request overhead.
- **No request size limit.** `read()` is capped at 4096 bytes and reads
  once. A request with a body larger than that will be truncated, but
  the server doesn't parse bodies at all (see Read-Only), so this is
  more an HTTP compliance note than a security issue.

## 🧾 Threat Model

Rservd assumes:
- The document root is owned by a trusted user (typically the one who
  wrote the pages), and is readable by `www`.
- The network is hostile — every byte from a client is untrusted.
- The kernel is trusted. Rservd does not implement sandboxing; it relies
  on FreeBSD's `setuid`/`setgid` and on filesystem permissions.
- The binary itself is trusted. Rservd is not hardened against binary
  tampering — if an attacker can replace `/usr/local/bin/rservd`, the
  game is over before it starts.

Rservd does **not** assume:
- That the client is honest.
- That the client speaks valid HTTP.
- That the client respects `rel="nofollow"`.
- That the request came from a "known good" IP.

## 📌 Summary

Three principles, applied consistently:

1. **Refuse to inform.** Every rejection is the same shape.
2. **Drop privileges early.** Bind, then drop, then serve.
3. **Check every path against the root.** Twice — `realpath` first, then
   `O_NOFOLLOW` on open.

The 403 page says *"Hold on there, pardner"* because the server's answer
to ambiguity is the same as the server's answer to attack: no.
