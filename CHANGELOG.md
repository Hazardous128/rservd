#Original: `c-httpd` from [tonybanters](https://github.com/tonybanters) | [video](https://youtu.be/RwSv24LpRJ0?si=MC1ws1APdFYt0Hc2) 
-----
- Serving pages (`socket()`, `bind()`, `listen()`, `accept()`, `read()`, `stat()`, `open()`)
- Path traversal fix (`the /.. bug`, `Fixing it with strstr`)
- Main mode (serving form ./www/)
- 404 page
- 4kb buffer
- Super small MIME type list (html, css, png, jpg; else text/plain)
- `/` index file (`/index.html`)
- Accepts `GET` request

#INFDEV: `rservd` - 26.11.14
-----
- Program name (retro server daemon)
- Directory listing
- compiled-in dynamic endpoints (/about:*)
- Thread-per-connection
- Enchanced file sending (`sendfile(2)` on FreeBSD (zero-copy), `read/write` fallback with 8 KB buffer elsewhere)
- Error pages (`403`, `404`, and almost empty `503`)
- Separated priveleges (Binds as `root`, drops to `www` before accepting requests)
- Per-request method + path to stdout; -v for verbose lifecycle messages (!kinda insecure!)
- Simplified compilation (makefile.md)
- Extended MIME types list 

Up to 128 "hand picked" file extensions on my pc (+some `application`s from apache mime.types)

### Includes:

- basic filetypes: images, audio, video
- (source) code and markup (mostly from FreeBSD sources):

| Group | MIME types|
|---|---|
| **Source code** | `c` ,`h` ,`cpp` ,`hpp` ,`cs` ,`rs` ,`mk` ,`patch` |
| **Markup + WebApps** | `js` ,`json` ,`htm(l)` ,`css` ,`php` ,`wasm` ,`node` ,`md` |
| **Configuration** | `cfg` ,`pem` ,`py` ,`lua` ,`jar` ,`sh` ,`rc` ,`nix` |

---

- archives: zip, rar, tar, gz, lz, bz, pkg (packages, xar), cw (classicube world, bz), lvl (minecraft/mcgalaxy world, gz)
- disk images: rom, img, iso, qcow, vbox, vdi ,vhd
- text: log, rtf, tmp, readme
- binary/executables: bin, so, exe, dll, bat

and thats not all

else `application/octet-stream`

- Enchanced security (more about it in current SECURITY.md file)
- Updated index page search (searching for index`.html`/`.htm` file for any `GET` request that got `/` on the end of the path)
