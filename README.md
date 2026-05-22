# sndr

[![Latest release][release-badge]][release-link]
[![Project license][repo-license-badge]](LICENSE)

[release-badge]: https://img.shields.io/github/v/tag/tarnover/sndr
[release-link]: https://github.com/tarnover/sndr/tags
[repo-license-badge]: https://img.shields.io/github/license/tarnover/sndr.svg

Easily and securely share files and directories from the command line through
a safe, private and encrypted link. Files are encrypted client-side before
upload; the decryption key never leaves the local machine unless explicitly
shared in the URL fragment. Anyone with the link can fetch and decrypt; the
server only ever sees ciphertext.

`sndr` is a maintained, security-hardened fork of
[`timvisee/ffsend`][timvisee-ffsend], itself derived from work on
[Mozilla's Firefox Send][mozilla-send] (discontinued in 2020). The fork
lineage:

```
mozilla/send  →  timvisee/ffsend  →  tarnover/sndr (this repo, rebranded)
```

`sndr` speaks the same protocol as the [`tarnover/send`][tarnover-send]
server (and any Send-compatible instance), so links produced by the server
can be uploaded to / downloaded from with this CLI as well as the browser.

## What's different in this fork

- **Rebranded `ffsend` → `sndr`** — binary, crate, env vars (`SNDR_*`),
  history path (`~/.cache/sndr/`), infer-command names (`sndrput`,
  `sndrget`, `sndrdel`). Existing ffsend dotfiles do **not** carry over
  — see [Migrating from ffsend](#migrating-from-ffsend).
- **Security hardening** — sanitises the server-supplied filename before
  joining with the user's output directory (prevents path traversal via
  metadata), enforces `0600` on every history-file save instead of only
  at create time, and fixes an inverted-condition bug that caused
  `sndr history forget` and expired-entry cleanup to silently skip the
  autosave.
- **`tarnover/send` short-URL support** — recognises `/dl/<id>` share
  paths and preserves the URL fragment across server-side `/download/`
  → `/dl/` redirects (so the decryption key survives reqwest's
  redirect-following).
- **Default Send host** — points at `https://snd.dx.pe/` (a
  `tarnover/send` instance) instead of the upstream default. Override
  with `--host` or `SNDR_HOST` to use any other Send instance.

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
- [Migrating from ffsend](#migrating-from-ffsend)
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

`sndr` uploads a file (or directory, archived on the fly) to a Send
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

---

## Quick start

```bash
# Simple upload
$ sndr upload my-file.txt
https://snd.dx.pe/dl/<id>/#<key>

# Advanced upload:
# - download limit of 1
# - expiry of 5 minutes
# - password-encrypt
# - archive (handy for directories)
# - copy the link to the clipboard
# - open the link in your browser
$ sndr upload --downloads 1 --expiry-time 5m \
    --password --archive --copy --open my-file.txt
Password: ******
https://snd.dx.pe/dl/<id>/#<key>

# Upload to a custom Send host
$ sndr u -h https://example.com/ my-file.txt
https://example.com/dl/<id>/#<key>

# Download
$ sndr download https://snd.dx.pe/dl/<id>/#<key>
```

Inspect remote files:

```bash
$ sndr exists https://snd.dx.pe/dl/<id>/#<key>
Exists: true

$ sndr info https://snd.dx.pe/dl/<id>/#<key>
ID:         b087066715
Downloads:  0 of 5
Expiry:     18h2m (64928s)
```

Manage your shares:

```bash
# View your share history
$ sndr history
#  LINK                                  EXPIRE
1  https://snd.dx.pe/dl/<id>/#<key>     23h57m
2  https://example.com/dl/<id>/#<key>   37m30s

# Set or change a password
$ sndr password https://snd.dx.pe/dl/<id>/#<key>
Password: ******

# Delete a share
$ sndr delete https://snd.dx.pe/dl/<id>/#<key>
```

Use `--help`, the `help` subcommand, or see [Commands](#commands) for the
full list.

---

## Migrating from ffsend

This is a full rebrand: no automatic migration runs at startup. If you are
coming from `ffsend`, here is what you need to do.

| What changes      | Old (`ffsend`)              | New (`sndr`)                 |
| :---------------- | :-------------------------- | :--------------------------- |
| Binary name       | `ffsend`                    | `sndr`                       |
| Env-var prefix    | `FFSEND_*`                  | `SNDR_*`                     |
| History file      | `~/.cache/ffsend/history.toml` | `~/.cache/sndr/history.toml` |
| Infer-command     | `ffput` / `ffget` / `ffdel` | `sndrput` / `sndrget` / `sndrdel` |
| Default host      | `https://send.vis.ee/`      | `https://snd.dx.pe/`         |

To carry your existing share history over once:

```bash
mkdir -p ~/.cache/sndr
cp ~/.cache/ffsend/history.toml ~/.cache/sndr/history.toml
```

To rename `FFSEND_*` env vars in your shell rc:

```bash
sed -i 's/FFSEND_/SNDR_/g' ~/.bashrc ~/.zshrc 2>/dev/null
```

You can keep `ffsend` installed alongside `sndr`; the two do not share
state once the history file is copied.

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

`sndr` is a young fork and is not yet published to crates.io or any
distro package index. Install from this repository:

```bash
# Install with cargo, straight from this fork
cargo install --git https://github.com/tarnover/sndr.git --locked

sndr --help
```

Or clone and build manually — see [Build](#build).

If you previously installed the upstream `timvisee/ffsend` binary (via
snap, AUR, Homebrew, MacPorts, Nix, Fedora, Alpine, FreeBSD ports,
Termux, or Docker), those packages still install the `ffsend` binary
under the upstream name and do **not** include this fork's hardening,
short-URL support, or `tarnover` defaults. They can coexist with
`sndr` — different binary names, different config paths.

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
git clone https://github.com/tarnover/sndr.git
cd sndr

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
| `infer-command`  |   ✓    | Honor `sndrput` / `sndrget` / `sndrdel` names      |
| `no-color`       |         | Disable colored error/help output                  |

Combine flags with `cargo install --no-default-features --features ...`.

### `sndrput`, `sndrget`, `sndrdel`

With the `infer-command` feature compiled in (default), symlinking the
binary under one of these names dispatches the corresponding subcommand:

| Link       | Equivalent           |
| :--------- | :------------------- |
| `sndrput`  | `sndr upload ...`    |
| `sndrget`  | `sndr download ...`  |
| `sndrdel`  | `sndr delete ...`    |

```bash
ln -s "$(which sndr)" ./sndrput
ln -s "$(which sndr)" ./sndrget
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

Invoke `sndr help <subcommand>` for per-command flags.

---

## Configuration and environment

`sndr` reads its defaults from environment variables. Each variable maps
to a CLI flag:

| Variable                | CLI flag                       | Description                            |
| :---------------------- | :----------------------------- | :------------------------------------- |
| `SNDR_HISTORY`          | `--history <FILE>`             | History file path                      |
| `SNDR_HOST`             | `--host <URL>`                 | Upload host (default `snd.dx.pe`)      |
| `SNDR_TIMEOUT`          | `--timeout <SECONDS>`          | Request timeout (`0` to disable)       |
| `SNDR_TRANSFER_TIMEOUT` | `--transfer-timeout <SECONDS>` | Transfer timeout (`0` to disable)      |
| `SNDR_EXPIRY_TIME`      | `--expiry-time <SECONDS>`      | Default upload expiry                  |
| `SNDR_DOWNLOAD_LIMIT`   | `--download-limit <DOWNLOADS>` | Default download limit                 |
| `SNDR_API`              | `--api <VERSION>`              | Server API version (`-` to autodetect) |
| `SNDR_BASIC_AUTH`       | `--basic-auth <USER:PASSWORD>` | Basic HTTP auth for protected proxies  |

Flag-style variables (presence sets the flag — value is ignored):

| Variable           | CLI flag        | Description                            |
| :----------------- | :-------------- | :------------------------------------- |
| `SNDR_FORCE`       | `--force`       | Skip warnings, force the action        |
| `SNDR_NO_INTERACT` | `--no-interact` | Disable interactive prompts            |
| `SNDR_YES`         | `--yes`         | Assume yes for prompts                 |
| `SNDR_INCOGNITO`   | `--incognito`   | Don't record actions in local history  |
| `SNDR_OPEN`        | `--open`        | Open the share URL after upload        |
| `SNDR_ARCHIVE`     | `--archive`     | Archive on upload                      |
| `SNDR_EXTRACT`     | `--extract`     | Extract on download                    |
| `SNDR_COPY`        | `--copy`        | Copy share link to clipboard           |
| `SNDR_COPY_CMD`    | `--copy-cmd`    | Copy a `sndr download` invocation      |
| `SNDR_QUIET`       | `--quiet`       | Minimal output                         |
| `SNDR_VERBOSE`     | `--verbose`     | Verbose logging                        |

Build-time variables:

| Variable     | Description                                                                |
| :----------- | :------------------------------------------------------------------------- |
| `XCLIP_PATH` | Hard-code the `xclip` binary path (with `clipboard-bin` on Linux / *BSD)   |
| `XSEL_PATH`  | Hard-code the `xsel` binary path (with `clipboard-bin` on Linux / *BSD)    |

There is no config-file support today.

---

## Scripting

`sndr` is designed for unattended use. The recipe:

- Always pass `--no-interact` (`-I`) — prompts that have no default will
  exit with an error rather than wait for a human.
- Pair with `--yes` (`-y`) and/or `--force` (`-f`) for actions you want to
  proceed regardless.
- Use `--quiet` (`-q`) when capturing the share URL into a variable.

Example:

```bash
set -e

# Upload, capture the URL
URL=$(sndr -Iy upload -q my-file.txt)

# Inspect (force, no prompts)
sndr -If info "$URL"

# Set a password
sndr -I password "$URL" --password="secret"

# Or apply the flags globally via env vars
export SNDR_NO_INTERACT=1 SNDR_FORCE=1 SNDR_YES=1

sndr download "$URL" --password="secret"
```

---

## Security

In short: `sndr` plus a trustworthy [Send][tarnover-send] instance is safe
for sharing sensitive files. Anyone holding the link can decrypt the
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

- Prefer `sndr` over a browser for sensitive shares — there is no
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
  removals (including `sndr history forget` and the auto-cleanup of
  expired entries on download) actually persist on autosave.
- `follow_url` preserves the URL fragment across HTTP redirects, so the
  decryption secret survives the `tarnover/send` `/download/` → `/dl/`
  301-redirect path.

No license, warranty, or guarantee comes with this — read the [LICENSE](LICENSE).

---

## Contributing

Pull requests and issues are welcome at
<https://github.com/tarnover/sndr>. Please keep changes focused, run
`cargo clippy --no-deps` and `cargo build` before submitting, and add
a test where it makes sense.

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
  [`timvisee/ffsend`][timvisee-ffsend] CLI that this fork was rebranded
  from.
- The contributors listed at upstream for keeping packaging, translations,
  and documentation healthy.

---

## License

This project is released under the GNU GPL-3.0 license.
See the [LICENSE](LICENSE) file for the full text.

[rust]: https://rust-lang.org/
[rustup]: https://rustup.rs/
[termux]: https://termux.com/
[timvisee-send]: https://github.com/timvisee/send
[send-encryption]: https://github.com/timvisee/send/blob/master/docs/encryption.md
