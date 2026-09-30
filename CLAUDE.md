# seekr

Terminal Soulseek client in Rust. The protocol layer and the TUI are written from scratch as a learning project.

## References
- Protocol spec: `docs/SLSKPROTOCOL.md` (local copy of the Nicotine+ protocol documentation). Treat it as the source of truth for message layouts. It is GPL-3.0, so it is git-ignored; get it with `scripts/fetch-protocol-doc.sh`.
- `michel/soulseek-rs` (MIT) is a **reference only**. Read it to see how tricky parts are solved (firewall piercing, distributed network), but do not add it as a dependency and do not copy code wholesale.

## Layout
- `crates/proto` (`seekr-proto`): pure message encoding/decoding with no I/O.
  - `wire.rs`: primitives (little-endian ints, length-prefixed strings with a Latin-1 fallback, IPv4).
  - `frame.rs`: `FrameCodec` splits the stream into payloads.
  - `server.rs`: server messages.
  - `peer_init.rs`: PeerInit / PierceFireWall and `ConnectionType`.
  - `peer.rs`: `PeerMsg` (one enum for both directions).
  - `search.rs`: `SearchResponse` / `SearchFile`, including the zlib body and attribute helpers.
  - Planned: `distrib.rs`, `file.rs`.
- `crates/net` (`seekr-net`): tokio networking.
  - `server.rs`: `ServerConnection` handles login and can be split into a reader and a writer.
  - `connect.rs`: opens direct and pierce connections, and `read_init` reads exactly one init frame.
  - `peer.rs`: the reader and writer tasks of a `P` connection.
  - `transfer.rs`: `receive` writes an `F` connection's data into `<file>.part`, resumes from an existing `.part` via FileOffset, and renames the file when done. `local_path` maps a remote path to `<download_dir>/<remote parent folder>/<name>`, sanitized.
  - `client/downloads.rs`: download bookkeeping for the actor (QueueUpload → TransferRequest → F connection). Until sharing exists, requests to download from us are answered with `File not shared.`.
  - `shares.rs`: `ShareIndex::scan` walks `shared_dirs` (hidden entries skipped) into virtual paths `<root name>\\rel\\path`. Audio properties come from lofty and are cached by path, size and mtime. It also provides search matching (all terms, `-exclude`) and browse listings.
  - `client/sharing.rs`: rescans, `SharedFoldersFiles`, and answers to server-relayed `FileSearch`, `SharedFileListRequest` and `FolderContentsRequest`.
  - `client/uploads.rs`: `QueueUpload` → queue → `TransferRequest` (upload) → `TransferResponse` → our `F` connection (`Purpose::Upload`) → token, the peer's FileOffset, file data. Slots are fair per user. `SendUploadSpeed` is sent after each upload.
  - `client.rs`: the actor. `Pending.purpose` says whether a connection is the user's `P` connection or an upload's `F` connection. It owns the server writer, the listener, the peer map and the pending connection attempts. Callers use `Client` and receive `Event`s.
- Peer connections follow the spec's "modern" order: `ConnectToPeer` and `GetPeerAddress` are sent together, and the direct and indirect attempts race keyed by token. If both fail, or 30s pass, a `PeerConnectFailed` event is emitted.
- Tokens for searches and connection requests come from one shared counter (`Tokens`). Search results whose token is not in `searches` are dropped.
- `crates/seekr`: the binary. Without a subcommand it starts the TUI (`src/tui/`):
  - `app.rs`: state and key handling.
  - `results.rs`: the search results as a folder tree; the cursor follows its row when results are re-sorted. `FormatFilter` (cycled with `f`/`F`) hides non-matching audio; non-audio files always stay visible.
  - `login.rs`: the login form.
  - `transfers.rs` / `uploads.rs`: the download and upload lists (`SpeedMeter` smooths speeds).
  - `../persist.rs`: `downloads.json`. Unfinished downloads are queued again on start (like Nicotine+).
  - `ui.rs`: rendering; only visible rows are built.
  - While the TUI runs, logs go to `~/.local/state/seekr/seekr.log`.
  - Render tests use `TestBackend` together with `Client::offline()`.
- CLI subcommands for testing: `login`, `userinfo <USER>`, `online`, `search <QUERY> [--full-paths]`, `shares [QUERY] [--dir D]`, `download <USER> <REMOTE PATH>`, `config-path`. The ratatui TUI will live here later.

## Conventions
- Request enums have `encode(&self, &mut BytesMut)`, which writes the full frame including the length prefix. Response enums have `decode(Bytes)`.
- Unknown message codes decode to `Unknown { code, payload }`; they never become errors.
- Every new message gets a unit test. Where the spec has a hex example, the test asserts the encoding byte for byte.
- Before finishing: `cargo fmt`, `cargo clippy --all-targets -- -D warnings`, `cargo test`.
- Log with `tracing`, to stderr for CLI commands and to a file once the TUI exists.

## Protocol constants
- Server: `server.slsknet.org:2242`. Default listen port: 2234.
- Major version 177, which the spec reserves for experimental development; minor version 1.
- Logging in with an unknown username **registers a new account**, so never test with made-up credentials.

## Config & login
- `~/.config/seekr/config.toml` (`src/config.rs`) holds `username`, `password`, and optionally `server`, `listen_port`, `download_dir` (default `~/Downloads/seekr`).
- It is written only through `write_private`, which writes a temp file and renames it (mode 600, directory 700), and it keeps unknown keys.
- Never commit it and never log credentials. `~/.config` is itself a public dotfiles repo, which ignores `seekr/`.
- First run: the TUI shows `tui/login.rs`, which checks the spec's username rules. Credentials are saved only after the server accepts them, and the file and folder are created at that point.
- With saved credentials, only a small "Connecting as …" splash appears. Network or port errors show the splash with `r` to retry. The form comes back only for INVALIDPASS or INVALIDUSERNAME. `seekr logout` clears the credentials.
- Password storage matches Nicotine+ and slskd (plain text in the config); ours has 600 permissions. Do not "encrypt" it with a key kept on disk. A keyring would be an opt-in feature.
- Settings tab (`tui/settings.rs`): `download_dir` is applied live through `Client::set_download_dir`. `shared_dirs` defaults to the XDG music dir (`~/Music`) and is only stored until milestone 6.

## Packaging
- `scripts/install.sh` installs to `~/.local/bin` (on PATH through `~/.config/fish/config.fish`). `scripts/uninstall.sh [--purge]` removes it; `--purge` also deletes config, data, state and cache (after confirmation) but never downloads.
- `packaging/aur/PKGBUILD` is a draft for a future AUR release; it is not published.
- CI (`.github/workflows/ci.yml`) runs fmt, clippy and tests on `main`.
- The README is user-facing and in English; keep its key table in sync with `help_line` in `tui/ui.rs`.

## Milestones
1. [x] Proto basics + login (`seekr login`)
2. [x] Listener + peer connections (PeerInit, ConnectToPeer / PierceFireWall)
3. [x] Search (FileSearch → zlib-compressed FileSearchResponse)
4. [x] Download (QueueUpload → TransferRequest/Response → F connection → FileOffset)
5. [x] ratatui TUI (search → pick → download, transfer list)
6. [~] Sharing: index, search answers, browse and uploads done; live test pending. Default share is `~/Music`, which does not exist on the dev machine. Searches mostly arrive through the distributed network (milestone 7), so until then only user and room searches reach us.
7. [ ] Distributed network
8. [ ] Extras: wishlist, browse, PMs, UPnP, MPRIS
9. [ ] AUR release, once stable
