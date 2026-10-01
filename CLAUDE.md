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
  - `shares.rs`: browse bodies (`SharedFileList`, `FolderContents`, both zlib-compressed).
  - File-connection messages (`FileTransferInit` = bare u32 token, `FileOffset` = bare u64) have no frame and are read and written directly in `net`.
  - `distrib.rs`: `DistribMsg` (u8 codes). `DistribSearch` must carry identifier 49. `decode_body` unpacks embedded messages.
- `crates/net` (`seekr-net`): tokio networking.
  - `server.rs`: `ServerConnection` handles login and can be split into a reader and a writer.
  - `connect.rs`: opens direct and pierce connections over a list of candidate addresses, and `read_init` reads exactly one init frame. `candidates` tries loopback first when a peer's IP equals ours (NAT hairpin), which also makes two local instances work.
  - `peer.rs`: the reader and writer tasks of a `P` connection.
  - `transfer.rs`: `receive` writes an `F` connection's data into `<file>.part`, resumes from an existing `.part` via FileOffset, and renames the file when done. `local_path` maps a remote path to `<download_dir>/<remote parent folder>/<name>`, sanitized.
  - `client/downloads.rs`: download bookkeeping for the actor (QueueUpload → TransferRequest → F connection).
  - `shares.rs`: `ShareIndex::scan` walks `shared_dirs` (hidden entries skipped) into virtual paths `<root name>\\rel\\path`. Audio properties come from lofty and are cached by path, size and mtime. It also provides search matching (all terms, `-exclude`) and browse listings.
  - `client/sharing.rs`: rescans, `SharedFoldersFiles`, and answers to server-relayed `FileSearch`, `SharedFileListRequest` and `FolderContentsRequest`.
  - `client/uploads.rs`: `QueueUpload` → queue → `TransferRequest` (upload) → `TransferResponse` → our `F` connection (`Purpose::Upload`) → token, the peer's FileOffset, file data. Slots are fair per user. `SendUploadSpeed` is sent after each upload.
  - `distrib_conn.rs`: read-only `D` connection to a (possible) parent; it closes when its handle is dropped.
  - `client/distrib.rs`: child-only membership in the distributed network. After login: `HaveNoParent(true)` and `AcceptChildren(false)`. `PossibleParents` leads to `start_connect_known` (`Purpose::ParentCandidate`). The first candidate that sends a search after its `BranchLevel` becomes the parent, and we report `HaveNoParent(false)`, `BranchLevel(level+1)` and `BranchRoot`. Server `EmbeddedMessage` means we are a branch root. `ResetDistributed` and losing the parent both mean searching again.
  - `portmap.rs`: UPnP IGD through `igd-next`. SSDP discovery goes **unicast to the default gateway first** (read from `/proc/net/route`), because ufw drops the reply to a multicast search (it comes from a different address). Multicast is the fallback. Mappings have a 1h lease, renewed every 30 min by the actor's `portmap_task`; they are unmapped when UPnP is turned off or the port changes. `PortInUse` counts as an existing manual forward. `behind_another_nat` flags a private or CGNAT WAN IP. The dev machine is double-NATed (the router's WAN is 192.168.100.2).
  - `client.rs`: the actor. `Pending.purpose` says whether a connection is the user's `P` connection or an upload's `F` connection. It owns the server writer, the listener, the peer map and the pending connection attempts. Callers use `Client` and receive `Event`s.
- Peer connections follow the spec's "modern" order: `ConnectToPeer` and `GetPeerAddress` are sent together, and the direct and indirect attempts race keyed by token. If both fail, or 30s pass, a `PeerConnectFailed` event is emitted.
- Tokens for searches and connection requests come from one shared counter (`Tokens`). Search results whose token is not in `searches` are dropped.
- `crates/seekr`: the binary. Without a subcommand it starts the TUI (`src/tui/`):
  - `app.rs`: state and key handling. Digits build a vim count (`App::count`) for the next motion (`10k`, `5G`), so tabs use `Alt-1…7` / `F1…F7` / `Tab`. The TUI starts with the results list focused; `s` or `/` open the search box (`/` on Browse edits the username instead), and `Alt-s` toggles box ↔ list from anywhere, since it types nothing.
  - `results.rs`: the search results as a folder tree; the cursor follows its row when results are re-sorted. `FormatFilter` (cycled with `f`/`F`) hides non-matching audio; non-audio files always stay visible. The filter survives new searches and browses (`Results::with_filter`), and titles show it with the filtered counts (`filter_note`). When it hides the cursor's row, the cursor moves to the folder of a hidden file, else the next shown folder, else the previous one. Lossless includes ALAC `.m4a` (via `is_lossy`). Folders sort case-insensitively. `shown_files` is kept incrementally; recounting it per burst made a 56k-file search 70× slower.
  - `login.rs`: the login form.
  - Browse tab (tab 5, also opened by `b` on a result): `Client::browse` → `Event::BrowseResult`. `share_list_as_response` turns the list into a `SearchResponse`, so the same `Results` tree, filter and download code serve both tabs (`Which::{Search, Browse}`). `render_result_list` draws either, with `show_user: false` for browse.
  - `transfers.rs` / `uploads.rs`: the download and upload lists (`SpeedMeter` smooths speeds).
  - `help.rs`: the `?` window. Per tab it shows what the tab is for, how it behaves and its keys, then the global keys; it swallows keys until Esc, `?` or `q`. **Keep it in sync** with `help_line` and the README key table when keys or behavior change. Search and Browse both start with their list focused (`/` or `s` opens the box), so `?` and the digits are never typed by accident.
  - `wishlist.rs`: Wishlist tab (tab 7). Saved queries run round-robin, one per server `WishlistInterval` (720s live, sent at login; default 12 min), and the first runs 30s after start. `App::tick` (every 500ms) fires them via `Client::wishlist_search` (server 103); results come back as normal `SearchResult`s and are matched by token. Each wish keeps its last 2 tokens and a `seen` set of `(user, file)`, so only unseen files count as new. `w` on Search results adds the query; `Enter` copies the wish's `Results` (which derive `Clone`) into the Search tab. Saved in `wishlist.json` (only the queries).
  - `buddies.rs`: Buddies tab (tab 6). Buddies are watched with `Client::watch_user` (server `WatchUser`); the answers (`WatchUser`, `UserStatus`, `UserStats`) arrive as `Event::ServerMessage`. The server pushes status changes but not stats, so `GetUserStats` is sent for every buddy whenever the tab is opened. `A` on a result, download or upload adds that user. Saved in `buddies.json` next to `downloads.json`.
  - `../persist.rs`: `downloads.json`. Unfinished downloads are queued again on start (like Nicotine+).
  - `ui.rs`: rendering; only visible rows are built. Tab titles carry no padding of their own (`Tabs` adds one space on each side). The header's right side only gets the width the tabs leave, and drops its least important parts first (sharing, then username, then speeds; online status stays), so 7 tabs still fit at 110 columns.
  - Quality strings (`src/search.rs`): kbps is always shown, estimated as `~N` from size and duration when the peer sends none. Lossless files also show sample rate and bit depth, and an `.m4a` above 600 kbps counts as ALAC, so it is lossless. Folder summaries use the minimum kbps for lossy folders and the average for lossless ones.
  - While the TUI runs, logs go to `~/.local/state/seekr/seekr.log`.
  - Render tests use `TestBackend` together with `Client::offline()`. `results.rs` has an ignored ingest benchmark (`cargo test --release -p seekr ingest -- --ignored --nocapture`).
- CLI subcommands (for testing and scripting): `login`, `userinfo <USER>`, `online`, `search <QUERY> [--full-paths] [--wishlist]`, `shares [QUERY] [--dir D]`, `browse <USER>`, `portmap`, `download <USER> <REMOTE PATH>`, `logout`, `config-path`.
- The actor sends `SetStatus(online)` after login and `ServerPing` every 60s.

## Conventions
- Request enums have `encode(&self, &mut BytesMut)`, which writes the full frame including the length prefix. Response enums have `decode(Bytes)`.
- Unknown message codes decode to `Unknown { code, payload }`; they never become errors.
- Regression tests must fail without the fix: when adding one, temporarily reintroduce the bug and watch the test fail. Fallback paths (for example, indirect connections) can hide failures of the main path, so assert the path too (`ConnectMethod::Direct`).
- `crates/net/tests/end_to_end.rs` runs a fake server on localhost with two real `Client`s (A shares, B downloads). Extend it for new peer or transfer flows; it also runs in CI.
- Every new message gets a unit test. Where the spec has a hex example, the test asserts the encoding byte for byte.
- Before finishing: `cargo fmt`, `cargo clippy --all-targets -- -D warnings`, `cargo test`.
- Log with `tracing`, to stderr for CLI commands and to a file once the TUI exists.

## Protocol constants
- Server: `server.slsknet.org:2242`. Default listen port: 2234.
- Major version 177, which the spec reserves for experimental development; minor version 1.
- Logging in with an unknown username **registers a new account**, so never test with made-up credentials.

## Config & login
- `~/.config/seekr/config.toml` (`src/config.rs`) holds `username`, `password`, and optionally `server`, `listen_port`, `download_dir` (default `~/Downloads/seekr`), `shared_dirs` (default the XDG music dir, `~/Music`) and `upload_slots` (default 2).
- Other files: `~/.local/share/seekr/downloads.json` (download list), `~/.local/share/seekr/buddies.json` (buddy names), `~/.local/share/seekr/wishlist.json` (wishlist queries), `~/.cache/seekr/shares.json` (audio property cache), `~/.local/state/seekr/seekr.log` (TUI log). Every path comes from `directories`, so XDG_* overrides work, which is handy for a second test instance.
- It is written only through `write_private`, which writes a temp file and renames it (mode 600, directory 700), and it keeps unknown keys.
- Never commit it and never log credentials. `~/.config` is itself a public dotfiles repo, which ignores `seekr/`.
- First run: the TUI shows `tui/login.rs`, which checks the spec's username rules. Credentials are saved only after the server accepts them, and the file and folder are created at that point.
- With saved credentials, only a small "Connecting as …" splash appears. Network or port errors show the splash with `r` to retry. The form comes back only for INVALIDPASS or INVALIDUSERNAME. `seekr logout` clears the credentials.
- Password storage matches Nicotine+ and slskd (plain text in the config); ours has 600 permissions. Do not "encrypt" it with a key kept on disk. A keyring would be an opt-in feature.
- Settings tab (`tui/settings.rs`) has the download folder, the listen port and the shared folders. The download folder applies live through `Client::set_download_dir`. The port uses `Client::set_listen_port`; it rebinds, sends `SetWaitPort`, and is saved only after an `Event::ListenPort` success. Changing the shared folders saves them and calls `Client::rescan_shares`.

## Interop testing
- `target/interop/` (git-ignored) holds an slskd 0.26.0 binary and a throwaway Soulseek test account (`slskd-account.txt`).
- Run seekr with XDG_* pointing into that folder and `listen_port = 2240`, and slskd headless with `--no-auth --http-port 5040 --slsk-listen-port 50301` (5031 collides with slskd's own HTTPS port). Drive it through `/api/v0/users/<u>/browse` and `/api/v0/transfers/downloads/<u>`.
- Delete the temporary seekr config afterwards, because it holds the user's password.

## Packaging
- `scripts/install.sh` installs to `~/.local/bin` (on PATH through `~/.config/fish/config.fish`). `scripts/uninstall.sh [-y]` removes everything (binary, config, data, state, cache) after one confirmation, but never touches downloads.
- `packaging/aur/PKGBUILD` is a draft for a future AUR release; it is not published.
- CI (`.github/workflows/ci.yml`) runs fmt, clippy and tests on `main`.
- The README is user-facing and in English; keep its key table in sync with `help_line` in `tui/ui.rs`.

## Milestones
1. [x] Proto basics + login (`seekr login`)
2. [x] Listener + peer connections (PeerInit, ConnectToPeer / PierceFireWall)
3. [x] Search (FileSearch → zlib-compressed FileSearchResponse)
4. [x] Download (QueueUpload → TransferRequest/Response → F connection → FileOffset)
5. [x] ratatui TUI (search → pick → download, transfer list)
6. [x] Sharing: index, search answers, browse and uploads. End-to-end tests cover direct, indirect and resume. Live-tested on 2026-09-30 on the real network against slskd 0.26.0 as downloader: browse parsed correctly, and a 3.3 MB FLAC arrived byte-identical (sha256). Default share is `~/Music`, which does not exist on the dev machine. Searches mostly arrive through the distributed network (milestone 7), so until then only user and room searches reach us.
7. [x] Distributed network, child only. Live-tested 2026-09-30: adopted a real parent within seconds and answered strangers' searches, and an slskd network-wide search found a probe file shared only by seekr. Next step: accept children and relay searches to them (needs a child connection manager and `AcceptChildren(true)`).
8. [~] Extras: browse done (live-tested: 28 713 files from a real user). UPnP is done (live-tested on the dev router, mapping plus renewal). Buddy list done (tab 6). Wishlist done (tab 7; live-tested: server sends WishlistInterval 720 and answers code-103 searches). Still to do: PMs, MPRIS.
9. [ ] AUR release, once stable
