# ffsend

[![Latest release][release-badge]][release-link]
[![Project license][repo-license-badge]](LICENSE)

[release-badge]: https://img.shields.io/github/v/tag/tarnover/ffsend
[release-link]: https://github.com/tarnover/ffsend/tags
[repo-license-badge]: https://img.shields.io/github/license/tarnover/ffsend.svg

Easily and securely share files and directories from the command line through
a safe, private and encrypted link. Files are encrypted client-side before
upload; the decryption key never leaves the local machine unless explicitly
shared in the URL fragment. Anyone with the link can fetch and decrypt; the
server only ever sees ciphertext.

This repository is a maintained fork of [`timvisee/ffsend`][timvisee-ffsend],
itself derived from work on [Mozilla's Firefox Send][mozilla-send]
(discontinued in 2020). The fork lineage:

```
mozilla/send  →  timvisee/ffsend  →  tarnover/ffsend (this repo)
```

`ffsend` speaks the same protocol as the [`tarnover/send`][tarnover-send]
server (and any Send-compatible instance), so links produced by the server
can be uploaded to / downloaded from with this CLI as well as the browser.

## What's different in this fork

- **Security hardening** — fixes a server-controlled path-traversal in the
  download path (a malicious uploader's filename could escape the chosen
  output directory), enforces `0600` on every history-file save instead of
  only at create time, and fixes an inverted-condition bug that caused
  `ffsend history forget` and expired-entry cleanup to silently skip the
  autosave. See commit
  [`2d31b6d`](https://github.com/tarnover/ffsend/commit/2d31b6d) for the
  diff.
- **Default Send host** — points at `https://snd.dx.pe/` (a
  `tarnover/send` instance) instead of the upstream default. Override with
  `--host` or `FFSEND_HOST` to use any other Send instance.

Everything else — the wire protocol, file format, command-line surface,
configuration knobs — is unchanged and remains compatible with downstream
Send servers and tooling.

[mozilla-send]: https://github.com/mozilla/send
[timvisee-ffsend]: https://github.com/timvisee/ffsend
[tarnover-send]: https://github.com/tarnover/send

---

## Table of contents

- [What it does](#what-it-does)
- [Quick start](#quick-start)
- [Requirements](#requirements)
- [Install](#install)
- [Build](#build)
- [Commands](#commands)
- [Configuration and environment](#configuration-and-environment)
- [Scripting](#scripting)
- [Security](#security)
- [Contributing](#contributing)
- [Acknowledgements](#acknowledgements)
- [License](#license)

---

## What it does

`ffsend` uploads a file (or directory, archived on the fly) to a Send
instance after generating a random key and encrypting the contents locally.
The URL it prints back to you contains the key in its fragment — the server
never sees it. Anyone with the link can fetch and decrypt; everyone else
sees only ciphertext.

Each share has a configurable expiry (default 24 hours, up to seven days on
most instances) and a download counter (default 1, up to 20). When either
runs out the ciphertext is deleted. Owners can password-protect a share,
change its download limit, or delete it themselves.

Features:

- Fully featured, scriptable command-line tool
- Upload and download files and directories, always encrypted on the client
- Optional password protection, passphrase generation, and configurable
  download limits
- File and directory archiving and extraction
- Share-URL shortener and QR code rendering
- Supports Send v3 (current) and v2
- Local history for managing your own shares
- Inspect, change parameters, or delete a share
- Streaming encryption/upload/download — low memory footprint, suitable
  for large files
- Designed for unattended use in scripts

[![ffsend usage demo][usage-demo-svg]][usage-demo-asciinema]
_No demo visible here? View it on [asciinema][usage-demo-asciinema]._

---

## Quick start

```bash
# Simple upload
$ ffsend upload my-file.txt
https://snd.dx.pe/#sample-share-url

# Advanced upload:
# - download limit of 1
# - expiry of 5 minutes
# - password-encrypt
# - archive (handy for directories)
# - copy the link to the clipboard
# - open the link in your browser
$ ffsend upload --downloads 1 --expiry-time 5m \
    --password --archive --copy --open my-file.txt
Password: ******
https://snd.dx.pe/#sample-share-url

# Upload to a custom Send host
$ ffsend u -h https://example.com/ my-file.txt
https://example.com/#sample-share-url

# Download
$ ffsend download https://snd.dx.pe/#sample-share-url
```

Inspect remote files:

```bash
$ ffsend exists https://snd.dx.pe/#sample-share-url
Exists: yes

$ ffsend info https://snd.dx.pe/#sample-share-url
ID:         b087066715
Name:       my-file.txt
Size:       12 KiB
MIME:       text/plain
Downloads:  0 of 10
Expiry:     18h2m (64928s)
```

Manage your shares:

```bash
# View your share history
$ ffsend history
#  LINK                                    EXPIRE
1  https://snd.dx.pe/#sample-share-url    23h57m
2  https://snd.dx.pe/#other-sample-url    17h38m
3  https://example.com/#sample-share-url  37m30s

# Set or change a password
$ ffsend password https://snd.dx.pe/#sample-share-url
Password: ******

# Delete a share
$ ffsend delete https://snd.dx.pe/#sample-share-url
```

Use `--help`, the `help` subcommand, or see [Commands](#commands) for the
full list.

---

## Requirements

- Linux, macOS, Windows, FreeBSD, or Android (other BSDs may work).
- A terminal and an internet connection.
- Linux:
  - OpenSSL and CA certificates
    (Debian/Ubuntu: `apt install openssl ca-certificates`).
  - Optional `xclip` or `xsel` for clipboard support.
- macOS / Windows:
  - Optional OpenSSL with the `crypto-openssl` build feature
    (the default `crypto-ring` backend needs no extra system packages).
- FreeBSD:
  - `pkg install openssl ca_root_nss` (and `xclip xsel-conrad` for the
    clipboard).
- Android:
  - Install via [Termux][termux].

---

## Install

The upstream `timvisee/ffsend` is published widely (cargo, snap, AUR,
Homebrew, MacPorts, Nix, Fedora, Alpine, FreeBSD ports, Termux, Docker).
Those packages do **not** include this fork's security fixes or the
`tarnover` default host. To run the hardened build today, install from
this repository's source. The packaged routes are listed below for
reference and for users who want the upstream binary instead.

### From this repository (recommended for the hardened build)

```bash
# Install with cargo, straight from this fork
cargo install --git https://github.com/tarnover/ffsend.git --locked

ffsend --help
```

Or clone and build manually — see [Build](#build).

### Upstream packages

The following install the upstream `timvisee/ffsend` build:

- **Linux (snap):** `snap install ffsend`
- **Arch (AUR):** `yay -S ffsend-bin` (binary) or `yay -S ffsend`
  (source)
- **Fedora:** `sudo dnf install ffsend`
- **Alpine:** `apk add ffsend --repository=http://dl-cdn.alpinelinux.org/alpine/edge/testing`
- **macOS (Homebrew):** `brew install ffsend`
- **macOS (MacPorts):** `sudo port install ffsend`
- **Windows (Scoop):** `scoop install ffsend`
- **FreeBSD:** `pkg install ffsend`
- **Android (Termux):** `pkg install ffsend`
- **Nix:** `nix-env --install ffsend`
- **Docker:** `docker run --rm -it -v "$PWD:/data" timvisee/ffsend ...`

For prebuilt binaries see the upstream
[GitHub releases][github-latest-release].

---

## Build

Build requirements:

- [Rust][rust] `1.63` (MSRV) or newer — install with [rustup][rustup].
- A C toolchain plus `pkg-config` and OpenSSL/LibreSSL headers when using
  the `crypto-openssl` feature; the default `crypto-ring` backend is
  pure-Rust and needs no system OpenSSL.
- Debian/Ubuntu example:
  ```bash
  sudo apt install build-essential cmake pkg-config libssl-dev
  ```

Build the project:

```bash
# Clone this fork
git clone https://github.com/tarnover/ffsend.git
cd ffsend

# Debug build
cargo build -j2

# Release build
cargo build --release -j2

# Install into ~/.cargo/bin
cargo install --path . -f
```

### Feature flags

| Feature          | Default | Description                                       |
| :--------------- | :-----: | :------------------------------------------------ |
| `send2`          |         | Support for Send v2 servers                       |
| `send3`          |   ✓    | Support for Send v3 servers                       |
| `crypto-ring`    |   ✓    | Pure-Rust crypto backend (no system OpenSSL)      |
| `crypto-openssl` |         | OpenSSL backend                                    |
| `clipboard`      |   ✓    | Copy share URLs to the clipboard                   |
| `history`        |   ✓    | Local share history                                |
| `archive`        |   ✓    | Archive directories on upload, extract on download |
| `qrcode`         |   ✓    | Render share URLs as QR codes                      |
| `urlshorten`     |   ✓    | Built-in URL shortener integration                 |
| `infer-command`  |   ✓    | Honor `ffput` / `ffget` / `ffdel` binary names     |
| `no-color`       |         | Disable colored error/help output                  |

Combine flags with `cargo install --no-default-features --features ...`.

### `ffput`, `ffget`, `ffdel`

With the `infer-command` feature compiled in (default), symlinking the
binary under one of these names dispatches the corresponding subcommand:

| Link    | Equivalent              |
| :------ | :---------------------- |
| `ffput` | `ffsend upload ...`     |
| `ffget` | `ffsend download ...`   |
| `ffdel` | `ffsend delete ...`     |

```bash
ln -s "$(which ffsend)" ./ffput
ln -s "$(which ffsend)" ./ffget
```

The full inferred-name table is defined in
[`src/config.rs`](./src/config.rs) under `INFER_COMMANDS`.

---

## Commands

| Command      | Aliases       | Description                            |
| :----------- | :------------ | :------------------------------------- |
| `upload`     | `u`, `up`     | Upload a file or directory             |
| `download`   | `d`, `down`   | Download a shared file                 |
| `info`       | `i`           | Show metadata for a share              |
| `exists`     | `e`           | Check whether a share still exists     |
| `parameters` | `params`      | Change download limit / expiry         |
| `password`   | `pass`, `p`   | Set, change, or remove a share password |
| `delete`     | `del`, `rm`   | Delete a shared file                   |
| `history`    | `h`           | List or forget locally-tracked shares  |
| `version`    | `v`           | Detect a Send server's API version     |
| `debug`      | `dbg`         | Print build and runtime debug info     |
| `generate`   | `gen`         | Generate shell completions / assets    |
| `help`       |               | Help for a subcommand                  |

Invoke `ffsend help <subcommand>` for per-command flags.

---

## Configuration and environment

`ffsend` reads its defaults from environment variables. Each variable maps
to a CLI flag:

| Variable                  | CLI flag                       | Description                            |
| :------------------------ | :----------------------------- | :------------------------------------- |
| `FFSEND_HISTORY`          | `--history <FILE>`             | History file path                      |
| `FFSEND_HOST`             | `--host <URL>`                 | Upload host (default `snd.dx.pe`)      |
| `FFSEND_TIMEOUT`          | `--timeout <SECONDS>`          | Request timeout (`0` to disable)       |
| `FFSEND_TRANSFER_TIMEOUT` | `--transfer-timeout <SECONDS>` | Transfer timeout (`0` to disable)      |
| `FFSEND_EXPIRY_TIME`      | `--expiry-time <SECONDS>`      | Default upload expiry                  |
| `FFSEND_DOWNLOAD_LIMIT`   | `--download-limit <DOWNLOADS>` | Default download limit                 |
| `FFSEND_API`              | `--api <VERSION>`              | Server API version (`-` to autodetect) |
| `FFSEND_BASIC_AUTH`       | `--basic-auth <USER:PASSWORD>` | Basic HTTP auth for protected proxies  |

Flag-style variables (presence sets the flag — value is ignored):

| Variable             | CLI flag        | Description                            |
| :------------------- | :-------------- | :------------------------------------- |
| `FFSEND_FORCE`       | `--force`       | Skip warnings, force the action        |
| `FFSEND_NO_INTERACT` | `--no-interact` | Disable interactive prompts            |
| `FFSEND_YES`         | `--yes`         | Assume yes for prompts                 |
| `FFSEND_INCOGNITO`   | `--incognito`   | Don't record actions in local history  |
| `FFSEND_OPEN`        | `--open`        | Open the share URL after upload        |
| `FFSEND_ARCHIVE`     | `--archive`     | Archive on upload                      |
| `FFSEND_EXTRACT`     | `--extract`     | Extract on download                    |
| `FFSEND_COPY`        | `--copy`        | Copy share link to clipboard           |
| `FFSEND_COPY_CMD`    | `--copy-cmd`    | Copy a `ffsend download` invocation    |
| `FFSEND_QUIET`       | `--quiet`       | Minimal output                         |
| `FFSEND_VERBOSE`     | `--verbose`     | Verbose logging                        |

Build-time variables:

| Variable     | Description                                                                |
| :----------- | :------------------------------------------------------------------------- |
| `XCLIP_PATH` | Hard-code the `xclip` binary path (with `clipboard-bin` on Linux / *BSD)   |
| `XSEL_PATH`  | Hard-code the `xsel` binary path (with `clipboard-bin` on Linux / *BSD)    |

There is no config-file support today; a dotfile-style configuration is
on the upstream roadmap.

---

## Scripting

`ffsend` is designed for unattended use. The recipe:

- Always pass `--no-interact` (`-I`) — prompts that have no default will
  exit with an error rather than wait for a human.
- Pair with `--yes` (`-y`) and/or `--force` (`-f`) for actions you want to
  proceed regardless.
- Use `--quiet` (`-q`) when capturing the share URL into a variable.

Example:

```bash
set -e

# Upload, capture the URL
URL=$(ffsend -Iy upload -q my-file.txt)

# Inspect (force, no prompts)
ffsend -If info "$URL"

# Set a password
ffsend -I password "$URL" --password="secret"

# Or apply the flags globally via env vars
export FFSEND_NO_INTERACT=1 FFSEND_FORCE=1 FFSEND_YES=1

ffsend download "$URL" --password="secret"
```

---

## Security

In short: `ffsend` plus a trustworthy [Send][tarnover-send] instance is
safe for sharing sensitive files. Anyone holding the link can decrypt the
contents, so the link itself is the secret to protect.

### Client-side encryption

Files and metadata are encrypted with `128-bit AES-GCM` before they leave
the machine; request authentication uses a `HMAC SHA-256` signing key.
The server never sees plaintext. The full ceremony is documented in the
[Send encryption notes][send-encryption].

### What's in the URL

The decryption secret lives in the URL fragment (`#...`), which browsers
do not send to the server. If you open a share link in a browser, however,
any JavaScript the server delivers can read the fragment. Mitigations:

- Prefer `ffsend` over a browser for sensitive shares — there is no
  server-shipped JavaScript in the loop.
- Use `--password` (during upload) or the `password` subcommand to add
  symmetric protection beyond the URL secret.
- Host your own Send instance (e.g. [`tarnover/send`][tarnover-send]).

### What this fork hardened

- Server-supplied filenames are reduced to their basename before being
  joined with the user's output directory, preventing a malicious
  uploader from writing outside the chosen destination via `..` or
  absolute paths.
- The history file (which contains owner tokens) is opened with
  `0o600` on every save, and `chmod 600` is re-applied after open, so a
  pre-existing file with looser permissions is fixed up rather than
  preserved.
- An inverted `changed`-flag check in `History::remove` is fixed so
  removals (including `ffsend history forget` and the auto-cleanup of
  expired entries on download) actually persist on autosave.

No license, warranty, or guarantee comes with this — read the [LICENSE](LICENSE).

---

## Contributing

Pull requests and issues are welcome at
<https://github.com/tarnover/ffsend>. Please keep changes focused, run
`cargo clippy` and `cargo build` before submitting, and add a test where
it makes sense.

If a change applies equally well to upstream `timvisee/ffsend`, please
open it there too so the broader community benefits.

---

## Acknowledgements

This project would not exist without:

- **Mozilla**, who designed and open-sourced the original
  [Firefox Send][mozilla-send] protocol and codebase.
- **Tim Visee** ([@timvisee](https://github.com/timvisee)), who kept the
  service alive after Mozilla discontinued it — both as the
  [`timvisee/send`][timvisee-send] server fork and as the original
  [`timvisee/ffsend`][timvisee-ffsend] CLI.
- The contributors listed at upstream for keeping packaging, translations,
  and documentation healthy.

---

## License

This project is released under the GNU GPL-3.0 license.
See the [LICENSE](LICENSE) file for the full text.

[usage-demo-asciinema]: https://asciinema.org/a/182225
[usage-demo-svg]: https://cdn.rawgit.com/timvisee/ffsend/6e8ef55b/res/demo.svg
[rust]: https://rust-lang.org/
[rustup]: https://rustup.rs/
[termux]: https://termux.com/
[timvisee-send]: https://github.com/timvisee/send
[send-encryption]: https://github.com/timvisee/send/blob/master/docs/encryption.md
[github-latest-release]: https://github.com/timvisee/ffsend/releases/latest
