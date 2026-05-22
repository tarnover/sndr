# sndr

[![Latest release][release-badge]][release-link]
[![License][license-badge]](LICENSE)

[release-badge]: https://img.shields.io/github/v/tag/tarnover/sndr
[release-link]: https://github.com/tarnover/sndr/tags
[license-badge]: https://img.shields.io/github/license/tarnover/sndr.svg

A command-line client for the **Send** end-to-end-encrypted file-sharing
protocol. Upload a file, hand someone a one-shot link, walk away. The
server stores ciphertext, the recipient holds the only key, the share
expires on its own.

```bash
$ sndr upload paper.pdf
https://snd.dx.pe/dl/8a3f.../#xY7q...
```

That URL is everything: file ID in the path, decryption key after the
`#`. Anyone who gets it can fetch and decrypt — and only them, because
the server never sees the key.

---

## Why use this

- **Send a file once, securely** — the alternative is "DM you a Google
  Drive link" (the operator holds your file forever) or "SFTP you onto
  my box" (you need an account and a network path). `sndr` is one
  command, no signup, no persistent footprint.
- **Encrypted before it leaves your machine** — AES-128-GCM with a
  random 128-bit key. The server stores opaque ciphertext.
- **One-shot by default** — 1 download, 24-hour expiry. Configurable
  up to 20 downloads / 7 days on most instances.
- **Optional password** on top of the URL secret, for cases where the
  link itself might leak.
- **Server-shipped JavaScript is out of the loop** — `sndr` ships its
  own implementation of the protocol, so a compromised server cannot
  swap out the crypto layer the way it could for a browser client.
  This is the real reason to use a CLI over the web UI for sensitive
  transfers.

It's not the right tool for everything. If you need persistent
hosting, multi-recipient access control, or files larger than a few
hundred MB on a public instance, use something else.

---

## Install

`sndr` is a young fork and is not yet packaged on crates.io or any
distro. Install from source:

```bash
cargo install --git https://github.com/tarnover/sndr.git --locked
sndr --help
```

Requires Rust `1.63` or newer. On Linux, you also want `ca-certificates`
(usually already installed), and `xclip` or `xsel` if you want
`--copy` to work. No system OpenSSL needed — the default crypto
backend is pure Rust.

For a hands-on build (cloning, debug build, testing locally),
see [Build](#build) below.

---

## Usage

### Upload

```bash
# The simple case
sndr upload paper.pdf

# 5 downloads, 1 hour expiry, password-protect, copy URL to clipboard
sndr upload --downloads 5 --expiry-time 1h --password --copy paper.pdf

# Upload a directory (auto-tars it)
sndr upload --archive ~/photos

# Read from stdin
tar c some/dir | sndr upload --name backup.tar -

# Different host
sndr upload --host https://send.example.com/ paper.pdf
```

### Receive

```bash
# Download to current directory, picking the original filename
sndr download "https://snd.dx.pe/dl/8a3f.../#xY7q..."

# Download to a specific path
sndr download -o /tmp/paper.pdf "https://snd.dx.pe/dl/8a3f.../#xY7q..."

# Inspect without downloading (does not consume a download slot)
sndr info "https://snd.dx.pe/dl/8a3f.../#xY7q..."
sndr exists "https://snd.dx.pe/dl/8a3f.../#xY7q..."
```

### Manage your shares

`sndr` keeps a local index of shares you've uploaded, so you can list,
re-share, or revoke them without re-pasting URLs.

```bash
sndr history                        # list your active shares
sndr password <url>                 # add or change the share password
sndr parameters <url> --downloads 2 # change the download cap
sndr delete <url>                   # revoke a share server-side
sndr history forget <url>           # drop a share from local history only
```

The history file lives at `~/.cache/sndr/history.toml` and contains
owner tokens, so it's stored with `0600` perms. Use `--incognito`
(or `SNDR_INCOGNITO=1`) to skip recording anything locally.

---

## Configuration

Every flag has an environment variable equivalent. Set what you use
most often in your shell rc.

| Variable                | Flag                           | Default              |
| :---------------------- | :----------------------------- | :------------------- |
| `SNDR_HOST`             | `--host <URL>`                 | `https://snd.dx.pe/` |
| `SNDR_HISTORY`          | `--history <FILE>`             | `~/.cache/sndr/history.toml` |
| `SNDR_EXPIRY_TIME`      | `--expiry-time <SECONDS>`      | 24 h                 |
| `SNDR_DOWNLOAD_LIMIT`   | `--download-limit <N>`         | 1                    |
| `SNDR_TIMEOUT`          | `--timeout <SECONDS>`          | 30 s                 |
| `SNDR_TRANSFER_TIMEOUT` | `--transfer-timeout <SECONDS>` | 24 h                 |
| `SNDR_BASIC_AUTH`       | `--basic-auth <USER:PASSWORD>` | _none_               |
| `SNDR_API`              | `--api <VERSION>`              | autodetect           |

Flag-style env vars (presence enables; the value is ignored):

```
SNDR_FORCE         SNDR_NO_INTERACT   SNDR_YES         SNDR_INCOGNITO
SNDR_OPEN          SNDR_ARCHIVE       SNDR_EXTRACT     SNDR_COPY
SNDR_COPY_CMD      SNDR_QUIET         SNDR_VERBOSE
```

There is no configuration file. Set env vars, or pass flags.

### Per-subcommand binaries

If you compile with the default `infer-command` feature, you can
symlink the binary under one of three special names and it dispatches
to the matching subcommand:

```bash
ln -s "$(which sndr)" /usr/local/bin/sndrput   # → sndr upload
ln -s "$(which sndr)" /usr/local/bin/sndrget   # → sndr download
ln -s "$(which sndr)" /usr/local/bin/sndrdel   # → sndr delete

sndrput file.pdf      # equivalent to: sndr upload file.pdf
```

---

## Security

### What the server sees

- Ciphertext bytes of your file.
- A few plaintext fields needed to route the share: file ID, nonce,
  owner-token hash, expiry timestamp, download counter, encrypted size.
- The IP address of every uploader and downloader.

### What the server cannot see

- The file's contents, filename, or MIME type. All three are
  encrypted client-side before upload.
- The URL fragment (`#...`), which is the decryption key. HTTP does
  not include the fragment in requests, and `sndr` is careful to
  preserve it across redirects.

### Threat model in one paragraph

`sndr` is safe to use over a hostile network: TLS protects the
ciphertext in flight, and the operator never holds the key. It is
also safe against a curious operator: they can see ciphertext but
not decrypt it. It is **not** safe against a compromised operator
who can swap the *web* client's JavaScript: when you open a share
link in a browser, the server's JS reads the fragment and could
exfiltrate it. `sndr` itself never runs server-supplied code, which
is the whole reason to prefer the CLI for sensitive shares.

The decryption key is in the URL. **Treat the URL like a password.**
Don't post it in a public channel. Don't paste it into a service that
keeps URL history. If you have any doubt, add `--password` so the
recipient also needs a passphrase you send through a different
channel.

### Hardening this fork carries vs. upstream `ffsend`

- Server-supplied filenames are reduced to their basename before
  being joined with the user's output directory — a malicious
  uploader cannot escape your target dir via `..` or absolute paths.
- The history file is reopened with mode `0600` on every save (not
  just at create time), defending against an attacker who pre-creates
  the file world-readable.
- An inverted-condition bug in `History::remove` was fixed; deletes
  and expired-entry cleanup now persist on autosave.
- `follow_url` preserves the URL fragment across HTTP redirects, so
  the decryption key survives the `tarnover/send` `/download/` →
  `/dl/` 301-redirect path.

For broader Send-protocol crypto details see the upstream
[encryption notes][send-encryption]. To report a vulnerability in
this fork, see [SECURITY.md](SECURITY.md).

---

## Scripting

`sndr` is designed for unattended use. The three flags you almost
always want in a script:

- `-I` / `--no-interact` — fail fast instead of waiting for stdin
- `-y` / `--yes` — assume yes for confirmations
- `-q` / `--quiet` — print only the share URL on upload

```bash
set -e
URL=$(sndr -Iy upload -q backup.tar.gz)
echo "share: $URL"
sndr -I password "$URL" --password="$(pwgen 24 1)"
```

Or set the equivalents globally and forget about them:

```bash
export SNDR_NO_INTERACT=1 SNDR_YES=1 SNDR_QUIET=1
```

`sndr` exits non-zero on any failure, including expired shares and
network errors, so `set -e` works as expected.

---

## Build

```bash
git clone https://github.com/tarnover/sndr.git
cd sndr

cargo build --release -j2          # build the release binary
cargo clippy --no-deps             # lint
./target/release/sndr --version

cargo install --path . -f          # install into ~/.cargo/bin
```

Feature flags (all enabled by default unless marked):

| Feature          | Default | What it does                                       |
| :--------------- | :-----: | :------------------------------------------------- |
| `send3`          |   ✓     | Send v3 protocol (current servers)                 |
| `send2`          |         | Send v2 protocol (older servers)                   |
| `crypto-ring`    |   ✓     | Pure-Rust crypto backend                           |
| `crypto-openssl` |         | OpenSSL crypto backend (needs system OpenSSL)      |
| `clipboard`      |   ✓     | `--copy` support via xclip / xsel / native API     |
| `history`        |   ✓     | Local share history                                |
| `archive`        |   ✓     | Tar directories on upload, untar on download       |
| `qrcode`         |   ✓     | Render share URLs as QR codes                      |
| `urlshorten`     |   ✓     | Built-in URL shortener integration                 |
| `infer-command`  |   ✓     | `sndrput`, `sndrget`, `sndrdel` symlink dispatch   |
| `no-color`       |         | Disable colored output                             |

Override with `cargo install --no-default-features --features ...`.

---

## Migrating from ffsend

If you already had `ffsend` installed and configured, here's the
quick migration. Nothing happens automatically — `sndr` is a clean
break, not a drop-in.

| What changes        | Old (`ffsend`)                 | New (`sndr`)                 |
| :------------------ | :----------------------------- | :--------------------------- |
| Binary name         | `ffsend`                       | `sndr`                       |
| Env-var prefix      | `FFSEND_*`                     | `SNDR_*`                     |
| History file        | `~/.cache/ffsend/history.toml` | `~/.cache/sndr/history.toml` |
| Symlink dispatch    | `ffput` / `ffget` / `ffdel`    | `sndrput` / `sndrget` / `sndrdel` |
| Default host        | `https://send.vis.ee/`         | `https://snd.dx.pe/`         |

Carry your share history over (one-time):

```bash
mkdir -p ~/.cache/sndr
cp ~/.cache/ffsend/history.toml ~/.cache/sndr/history.toml
```

Rename env vars in your shell rc:

```bash
sed -i 's/FFSEND_/SNDR_/g' ~/.bashrc ~/.zshrc 2>/dev/null
```

`sndr` and `ffsend` can coexist — different binary names, different
config paths — so you can roll back at any time by switching back to
the old binary.

---

## Project status

This fork is maintained by [tarnover](https://github.com/tarnover) to
support the [`tarnover/send`](https://github.com/tarnover/send) server
fork. Upstream [`timvisee/ffsend`](https://github.com/timvisee/ffsend)
remains the canonical implementation for users who want the
broadly-packaged binary.

```
mozilla/send  →  timvisee/ffsend  →  tarnover/sndr (this repo)
```

Bug reports and PRs welcome at
[github.com/tarnover/sndr](https://github.com/tarnover/sndr). See
[CONTRIBUTING.md](CONTRIBUTING.md) for the contribution flow and
[SECURITY.md](SECURITY.md) for vulnerability reporting.

---

## Acknowledgements

- **Mozilla** — designed and open-sourced the original
  [Firefox Send][mozilla-send] protocol and codebase before
  discontinuing the service in 2020.
- **Tim Visée** ([@timvisee](https://github.com/timvisee)) — kept the
  protocol alive as both the [`timvisee/send`][timvisee-send] server
  fork and the original
  [`timvisee/ffsend`][timvisee-ffsend] CLI that this rebrand started
  from.

---

## License

GNU GPL-3.0. See [LICENSE](LICENSE).

[mozilla-send]: https://github.com/mozilla/send
[timvisee-ffsend]: https://github.com/timvisee/ffsend
[timvisee-send]: https://github.com/timvisee/send
[tarnover-send]: https://github.com/tarnover/send
[send-encryption]: https://github.com/timvisee/send/blob/master/docs/encryption.md
