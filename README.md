Sources & playlist formats
Pretty much anything you throw at it works as a source:

M3U / M3U8 — local file, URL, USB stick, SMB share, WebDAV
Xtream Codes API — Live, VOD and Series as independent toggles per account, with category filtering
Stalker / MAG portal (MAC-style) — with automatic fallback across the common portal paths
TVheadend — as a native playlist type
SMB (jcifs-ng) and WebDAV — full file browsing, recursive scan, used as a real source not just a download target
Built-in M3U editor — fully browser-based, paginated, batch-rename, custom-logo override, per-channel visibility, per-profile reset, restore-original
Playlist combinator — merges multiple sources into one virtual list with combined EPG
Per-playlist EPG offset, custom HTTP headers, custom User-Agent per channel
Auto-refresh per playlist on its own interval, optional refresh-on-startup

M3U / KodiProp / DRM
This was the biggest pain point in other players for me, so I went deep:

Full KodiProp / inputstream.adaptive.* property set — license_key, license_type, license_data, manifest_type, drm_legacy, stream_headers, manifest_headers, max_bandwidth, min_bandwidth, video_resolution_max, max_width, max_height, audio_language, subtitle_language, live_delay, and the rest of the IA-Adaptive surface. Anything that works in Kodi works here too.
ClearKey in every form I've encountered in the wild: hex KID:Key pairs, hex with dashes or colons, base64-URL JWK JSON, plain base64 JWK, data:application/json;base64,… Data-URIs, multi-key JWK containers for multi-DRM streams. Auto-normalizes padding, fixes kty/k/kid, injects type:"temporary" where missing.
PSSH generator — when a stream provides KIDs but no PSSH box, a valid ClearKey PSSH is synthesized on the fly so the CDM can issue a key request.
Widevine + PlayReady via JSON drm_configs exactly like IA-Adaptive expects (server_url, req_headers, init_data), with Widevine L1 hardware paths on devices that support it.
Widevine L1 Manifest Patcher — for the awkward L1 streams: injects missing BaseURLs, propagates query tokens onto every initialization= and media= segment with smart de-dup, escapes naked & characters that crash the parser, and detects signed CDN URLs (common signature schemes with sig+exp, AWS-style Signature/Policy/Key-Pair-Id, CDN-token formats with hdnea/hdntl, generic auth/token/expires) and switches to single-DRM-session mode plus a desktop-Chrome User-Agent so the signed CDN actually serves the file.
Two independent MPD rewriters cover Widevine, PlayReady, FairPlay, MP4Protection ContentProtection blocks, bandwidth cap injection, dynamic ClearKey AdaptationSet injection — auto-picked per stream.
Smart Proxy Engine — a per-stream local HTTP proxy that resolves redirects, sniffs the real CDN URL, harvests DRM data after a redirect, replays session tokens, and caches sessions (with cache-bypass for signed URLs that expire). Includes an AudioTrackProber that detects dead audio renditions (e.g. mp4a + AC-3/EC-3 mixes) and skips them before playback errors.
Trust-All TLS for streams — optional OkHttp stack that accepts self-signed/expired certs (extremely common with IPTV providers), while the activation server uses a strictly pinned client.
Manifest formats detected and auto-handled: DASH (.mpd), HLS (.m3u8), MPEG-TS, Smooth Streaming, MP4/MKV/AVI/MOV/M4V/MPEG/MPG, WebM (VP8/VP9/AV1), M2TS/MTS (Blu-ray, AVCHD), VOB (DVD-Video), 3GP/3G2, WMV/ASF, FLV/DIVX/OGV/OGM/RM/RMVB. Auto-detect by extension, query-string format hint, explicit manifestType field, or redirect-follow pre-flight for shortener links.

Playback
ExoPlayer / Media3 latest, hardware-decoder-first with software fallback, codec-fallback chain, mixed-MIME adaptive switching.

Audio passthrough for AC-3, E-AC-3, E-AC-3-JOC (Atmos JOC), DTS, DTS-HD, TrueHD — bitstream straight to the receiver. E-AC-3 (Dolby Digital Plus) toggle that switches the extension-renderer mode.
3-band broadcast-grade Audio Normalizer — Bass (≤250 Hz) at 3.5:1 with slow attack/release against pumping, Mids/Dialog (250–4000 Hz) at 4.5:1 with wide soft-knee and +4 dB make-up to lift dialog, Highs (>4 kHz) at 2.5:1 against harsh sibilance, brick-wall limiter at -1 dB, noise gate at -50 dB with gentle expansion. Tuned to EBU R128 / ATSC A/85 broadcast standards. Adapts to detected channel count (mono → 7.1) and pops a conflict dialog if you have passthrough enabled.
Loudness EQ — separate "make quiet channels louder" boost, independent of the normalizer.
Audio Delay Processor — per-channel audio delay ±1000 ms in 50 ms steps, applied through a custom Media3 AudioProcessor (circular buffer for positive delay, byte-skip for negative).
Audio Sync Manager — per-channel persistent audio offset, session-only or permanent storage, full backup/restore of all offsets.
Frame-Rate Manager (AFR) — 4-stage system: Surface.setFrameRate() on Android 11+, Display.Mode matching with a scoring algorithm (exact match, integer multiples, 3:2 pulldown), ExoPlayer seamless frame-rate-strategy fallback, and an adaptive drop-monitor that re-evaluates the display mode when too many frames drop. Recognizes 23.976, 24, 25, 29.97, 30, 48, 50, 59.94, 60, 100, 120 fps plus exotic 12/15/20 fps. Normalizes encoder mis-reports (23.98 → 23.976 etc.) and listens to external display-mode changes via DisplayManager.
HDR-to-SDR tone mapping toggle for displays that need it.
Video tunneling toggle for low-latency on capable TV decoders.
Strict hardware-decode toggle (disables the software fallback path).
Smart Local Timeshift — pause live TV, resume from buffered cache, dynamic cache-size mode that bumps buffer while paused and shrinks back near the live edge. "Back to live" pill, time-since-pause counter, pause overlay. Disabled gracefully on DRM streams.
Live edge control — configurable distance to live (Default, 10 s, 20 s, 30 s, 60 s).
Per-hardware-class buffer tuning — three hardware classes (weak TV stick, mid-range, premium). Each gets its own load-control profile, start buffer (300 ms to 2000 ms), rebuffer threshold, min/max buffer windows. Live TV uses tighter buffers than VOD.
Fast Zapping Mode — aggressive buffer profile for sub-second channel switches.
Playback Watchdog — detects stalled streams and force-reloads with exponential backoff (5 s → 15 s → 60 s → 180 s cap) so dead streams stop hammering the server. Auto-resets after a healthy playing period, knows to leave portal-style and Xtream URLs and VOD alone.
ExoPlayer re-use across zaps for fast switching, with a max-reuse safety net and signed-URL detection that forces a rebuild before tokens expire.
Auto-Failover — switches to a backup stream variant when the primary dies.
Stream Health overlay — live readout updated every second: resolution, codec, real-measured bitrate (via AnalyticsListener byte counters, not manifest claims), FPS, dropped frames, buffer level, audio format, container format. Draggable panel.
Aspect ratio modes — Original, Fit Width, Full Screen, Zoom, Auto.
Channel zapping modes — Default, Fade, Fade with audio crossfade.
MultiView — Picture-in-Picture or Split-Screen with two channels at once. Deferred audio swap (audio stays on the old tile until the new tile is buffered, no audio glitches), tap to swap audio focus, animated borders, PiP-corner cycling, surface switch via PlayerView.switchTargetView() for ruckelfree handoff. Blocked gracefully on DRM streams.

Catchup / Replay
Full multi-format support, any combination works:

All common template variables: {utc}, {lutc}, {utcend}, {start}, {end}, {now}, {duration}, {durmin}, {timestamp}, {offset}, {Y}/{m}/{d}/{H}/{M}/{S} plus matching {*end} variants, {iso}, {isoend}, {date}, {xctime}, {catchup_id}, {stream_id}, {url} — both {var} and ${var} syntax accepted
Append / Shift / Default style ?utc=…&lutc=…
Generic Timeshift ?timeshift=offset
Xtream/XC Timeshift with auto-extracted username/password/streamId
Flussonic HLS (timeshift_abs-START.m3u8)
Flussonic TS (timeshift_abs-START.ts)
Portal-style catchup (resolved on demand via the portal's createCatchupLink)
NPVR (archive=START&archive_end=END)

Catchup streams are tagged as VOD so D-Pad seek, scrub bar and resume controls activate automatically.
EPG
Built a custom binary EPG engine because the existing approaches were too slow for big playlists:

Lazy-loading binary EPG — custom on-disk format with header, body and footer-index. Cold start reads only the small footer, builds an xmlId → (offset, length) lookup, then memory-maps the body on demand. No more multi-second EPG load on cold start — works for playlists of any size.
mmap-based per-channel parse, generation counter for instant cache invalidation, LRU cache size per hardware class, live-overlay map for fresh sync data before it's persisted, micro-cache for "current program" lookups so subsequent zaps are free, background-priority I/O thread pool with idle-gate so saves never block playback, adaptive chunked DB writes that shrink during zapping and grow when idle.
Robust XMLTV parser — sanitizes XML-1.0-illegal control bytes that crash other parsers, auto-detects encoding from the prolog (UTF-8, ISO-8859-1, …) so umlauts never mojibake, ten date-format parsers including PAL/NTSC, ISO, Kodi/XMLTV timezones, with a "memorize last successful format" cache for huge speed-ups on big EPGs. Per-block recovery: if one <programme> is corrupt, only that block is discarded.
EPG source variety — XMLTV plain, XMLTV gzipped, multiple URLs merged, Xtream API EPG (xmltv.php + per-stream getStreamEpg), portal-style EPG (per-channel, per-day, parallel with semaphore), M3U-embedded EPG URL auto-extracted, manual standalone EPG URL.
Rich metadata — title, description, icon URL, categories, year, episode-num, age rating, star rating, director, actors, country. Flags: NEW, PREMIERE, PREVIOUSLY-SHOWN (repeat), LAST CHANCE — surfaced as little markers in the info strip.
First-run fast-path — pushes live "now playing" entries to the UI as they're parsed, before the chunk flush, so a fresh sync shows channel info instantly.
Smart EPG matching (Deep Match) — fuzzy matching that finds the right XMLTV channel even when M3U tvg-id and XMLTV channel id look nothing alike. Strips sub-variant suffixes (e.g. channelname_extras → channelname, channelname_hd2 → channelname), country-code prefixes in five formats (DE:, [DE], (DE), DE |, DE - ) with whitelisted ISO codes so legitimate channel names never get mis-stripped. Strips promo noise (vip, premium, 4k, hevc, 1080p, dolby, atmos, …) and multilingual region tokens. Converts symbols to words (+ → plus) so names with special characters still match. Built-in alias dictionary, O(log N) TreeMap prefix search, emoji/Unicode strip before alias lookup.
Persistent Artwork Cache — disk-cached cross-EPG artwork lookups, RAM-mirrored on app start.
EPG Negative Cache — remembers which XMLTV IDs have no data, so re-queries are free instead of running another backfill. Invalidates automatically when a new sync changes the generation token.
Manual EPG Assignment — pick any channel, assign any XMLTV ID from any source, searchable with alphabet-jump sidebar.
Pre-warmed Channel/EPG Cache for currently focused groups, ready before the user zaps.
Cross-EPG Artwork Pickup — if the primary EPG has no artwork for a program, the player uses artwork from a secondary source that does.
EPG-driven Sleep Timer — "off after current program" mode based on EPG end time.
Rich UI — Now/Next grid, vertical Short-EPG, full Grid EPG, Full EPG details — all switchable. Metadata strip with categories, year, country, episode, age rating, star rating, NEW/PREMIERE/REPEAT/LAST CHANCE markers, director and cast line.

Recordings & Series

Native FFmpeg backend — custom-built FFmpeg shim that replaces the bulky ffmpeg-kit dependency. Same API, around 10 MB smaller APK. Supports async execute, cancel, log/stats callbacks, FFmpeg pipes.
Multi-target recording — internal storage, USB, SMB share, WebDAV.
DASH local-proxy for recording — internal HTTP proxy that rewrites segment URLs and replays custom headers so FFmpeg gets a "normal" stream to record from, even on DRM/DASH content.
Series Timer (auto-record) — rule-based: search pattern + match mode (Contains/Starts with/Exact), channel filter, day-of-week bitmask, time-window, priority for conflict resolution, max-keep auto-rotate, auto-delete-after-X-days, per-profile rules.
Recording metadata — every recording stores the full EPG snapshot at record time (start/end, icon, category, year, episode, rating, star rating, director, actors, country), file path, storage type, custom name, last-played position for resume, duration, file size, original channel link. Survives EPG roll-off.
Recording Registry — live & scheduled overview, filtering (All / Active / Scheduled), pause/resume/edit/delete, partial-save guarantee on cancel.
Recording-aware UI — pre-record countdown prompt, REC/STOP overlay buttons, "running" badge on the affected channel.
History sub-group for recordings, sorted by last-played-time, long-press to remove single entries.

Network & Privacy

Per-Playlist WireGuard VPN — real WireGuard-for-Android backend, multiple profiles, AES-256-GCM-encrypted profile storage (AndroidX Security Crypto + EncryptedSharedPreferences), per-playlist auto-connect, automatic profile switching when zapping between playlists, in-app .conf import.
Zero-Cloud, Zero-Tracker — no telemetry, no analytics, no third-party SDKs, no crash reporters, no ads. Ever. Full GDPR/privacy statement.
Encrypted Support Log — optional support-log toggle (default off, auto-disables after 5 minutes). Strips every URL, IP, MAC and credential through an irreversible filter before any line is written. On manual export, packs the ring buffer into a .vlog file using RSA-3072 key wrapping + AES-256-GCM. Only my private key can decrypt it.
Encrypted Backups — VIPBACKUP:3 format: AES-256-GCM with 128-bit auth tag, PBKDF2WithHmacSHA256 with 310 000 iterations (OWASP-compliant), 32-byte random salt + 12-byte random IV per backup, HMAC-SHA256 over salt+IV+ciphertext with a separately-derived key. Header stores the iteration count so older backups still decrypt. Backwards-compatible with legacy unencrypted backups. Per-app passphrase from a native library — works on any device with the app, no manual password needed. Backup targets: local file, USB, SMB, WebDAV.

Channel Preview & Thumbnails

Live channel preview in the list — a small live player instance loads the focused channel so you see a live mini-preview while scrolling. Debounced, skips DRM/L1/protected streams that only allow one concurrent session, skips VOD/SMB/WebDAV/recordings.
Live Preview Cache — when you zap away from a channel, the last visible frame gets grabbed as a tiny JPEG, saved to disk (per-hardware-class) and RAM-LRU'd. Next scroll through the list, every channel you've visited shows its own micro-preview instead of just a logo. Dedicated HandlerThread for disk I/O so the hot path stays pure RAM.
Background Preview Capture — for VOD/Movies, the player starts the stream, seeks a few minutes in, captures a single frame and uses it as the EPG/VOD background image — independent of the main player.
Frame Capture Engine — three-strategy frame extraction: TextureView.getBitmap() (fastest), PixelCopy on SurfaceView (handles DRM-permitted compositor frames), View.draw() UI fallback. Detects all-black/blocked frames and discards.
Movie Thumbnail Harvester — for SMB/WebDAV movies without a poster, auto-grabs a frame a few minutes in (PixelCopy, JPEG, 80% quality) and uploads it to a meta/ sub-folder next to the video. SMB/WebDAV scanners pick it up next sync. Orphan cleanup (deletes meta files whose movie is gone).
Background Preview Image — faint full-screen background behind EPG/VOD text using the program's iconUrl or a cross-EPG-matched artwork; falls back to the cached frame.

Remote Web Dashboard
Built-in HTTP server (configurable port, default 8888) with optional HTTPS. Open the device's local IP in any browser:

PIN-based auth (SecureRandom 6-digit, visible on the TV), 30-minute session timeout, 5-attempts-then-5-minute-lockout per IP, rate limiting, 32 MiB body limit.
Network detection — LAN IP, public IP, CGNAT/DS-Lite detection, double-NAT warning, port-forward hint, HTTPS port advertisement.
Per-dashboard-session profile — the dashboard user can edit a different profile than the TV is currently playing, so admin tasks don't disturb the family's active session.
Exposed in the dashboard — read/write settings, structured settings, key bindings, playlists CRUD, themes, EPG sources, playlist reload, EPG reload, backup export/import, local/SMB/WebDAV backup save+restore+list+restore-remote, support-log status/export, source-groups, groups per playlist, channels per group, channel rename, custom logo set, group rename, names reset, VPN profiles CRUD, EPG coverage, EPG-ID list, EPG assign, Kids PIN set/clear, profile CRUD (create/edit/delete/switch/active), full M3U editor (groups/channels/series/episodes/save/import/export/reset-group/reset-channel/restore-original), recordings (list/rename/delete), playlist reorder.

UI / Personalization

Multi-Profile — multiple profiles with their own favorites, channel/group visibility, watch history, series rules, recordings, resume positions, custom renames and settings. Playlists and EPG are shared. "Who's watching?" picker with avatar colors.
Kids Mode — PIN set by 4 D-Pad direction presses on the remote, age-rating filter, per-playlist allowed-groups list, dedicated Kids Profile type with auto-filtered adult content, blocks profile switching while active.
30 themes — Classic Blue, Emerald, Amber Gold, Crimson, Midnight Violet, Steel, Ocean Deep, Copper, Slate Blue, Forest, Rosé Gold, Nebula, Burgundy, Sunrise, Arctic, Sage, Magma, Sapphire, Sakura, Carbon, Titanium, Lavender, Mocha, Neon Mint, Obsidian, Ruby, Aurora, Sand, Electric, Phantom — each with primary, dark, soft, focus, progress, success/warning/error, surface, dialog, item, focus-item, text-primary/secondary/hint and stroke colors. Every accent widget (button, slider, EditText, progress bar, spinner) re-skins instantly.
Custom Color Overrides — on top of themes, individually overridable text colors for group list, channel list, current program, upcoming program, start time, detail text, channel overlay, current-program-in-overlay, times/clock — with a curated muted palette. Reset-to-theme-defaults at any time.
Transparency sliders — independent overlay and dialog transparency, plus overlay/highlight corner rounding, text outline toggle.
Channel Overlay Bubble — bottom-of-player info bubble with channel name, current program, start/end times, next program, REC/STOP indicators, optional artwork. Every text element individually colorable.
Global font-size profile — Small / Medium / Large / Custom, affects sidebar, channel names, program titles, preview list, detail text, overlay.
Auto-Hide system — master switch plus per-element timeouts for: Channel Info Overlay, Full EPG Details, Quick Actions, Full EPG View, Menu/Channel List, Track Selection Dialog, Sleep Timer Dialog, Audio Sync Dialog, Mesh Sync Prompt, Auto-Switch Countdown, Recording Countdown.
Smart Radar (live sports radar) — tracks user-defined keywords (your favorite teams, leagues, athletes, motorsport series, whatever) and surfaces matching live programs across all channels as a synthetic "now live" group at the top of the channel list. Never miss kick-off.
Voice Search (beta) — Google speech recognition with channel-name and program-name matching, "Switching to: …", "Playing: …", graceful offline fallback.
Spotlight-Style Search — searches across Channels, Programs, Movies/VOD, Upcoming Programs simultaneously, with LIVE/VOD/SERIES/EPG result tags, recent-searches history, instant filter as you type.
Custom Key Bindings — every D-Pad direction, OK button, channel up/down, menu key — short and long press — re-mappable to Zap Up, Zap Down, Open Menu, Last Channel, Channel Info, Full EPG, Quick Actions, Play/Pause, Seek Forward, Seek Backward, or None.
Channel numbers — optional channel numbers, number-pad zapping on the remote, tvg-chno sort, 5-second number-entry buffer.
Inverted D-Pad for zapping.
Group-end loop — wrap around at the end of a channel list.
Edit modes — channel-visibility (click-to-hide), group-visibility, drag-handle group sort/move, per-group "restore all channels in this group", "restore all groups".
History group — virtual "History" group with per-content-type sub-groups, configurable max-entries, long-press to remove single entries.
Favorites group — mark-with-OK mode, per-profile, persistent across sessions.
Resume dialog — for VOD/Series/Recordings, "Continue from <time>" or "From start", auto-detection of next episode.
Logo pipeline — logo cache toggle, B&W logo toggle (greyscale all logos), prefer-XML-logos, prefer-M3U-logos (per playlist), custom logo per channel (URL or local file via SAF), global logo path fallback (local SAF folder or HTTPS base URL with auto-match by normalized name), logo size slider.
Touch gestures — edge-swipe brightness/volume with OSD, tap-to-show-controls, double-tap seek.
High-performance UI — smooth even on weak Android TV boxes, dynamic hardware scaling that adapts UI complexity to device specs.
37 languages localized.

Security & Activation

Native secrets library — all crypto secrets (AES key, IV, HMAC key, backup passphrase) live in a native shared library. Reverse engineers need Ghidra/IDA + ARM-ASM instead of jadx.
AES-256-CBC + HMAC-SHA256 for activation payloads, HMAC verified before decryption.
Encrypted token storage — AndroidX Security Crypto with MasterKey.AES256_GCM keystore + EncryptedSharedPreferences (AES256-SIV for keys, AES256-GCM for values).
Anti-tamper / Root / Emulator / Hook detection — checks for su binaries across 16 known paths, common root manager packages, dangerous props, read-write /system mount checks, which su execution, emulator detection by Build properties, instrumentation framework traces, APK signature verification.
SSL pinning for the activation server (trust-all only for stream endpoints).
Offline grace — 3 days of offline use allowed before re-check.

Other internals

Stable Channel IDs — FNV-1a 64-bit hash of (playlistId, name, group, dupIndex) so channel IDs stay constant even if the provider rotates URLs/CDN/tokens. Favorites, audio sync, series rules and history all survive server moves.
Hardware-class system — three tiers (weak TV stick, mid-range, premium). Buffer sizes, LRU capacities, cache caps, animation enable/disable, chunk sizes, save throttling, idle gates — all auto-tune.
Adaptive DB-write chunking that shrinks during user activity and grows when idle so the UI never freezes during a large EPG ingest.
Stream-idle watcher — global gate that throttles every background save/I-O job so it never collides with playback.
Mesh Sync — mDNS-based peer discovery across the LAN. Find paused streams on other TVs in the house, resume them on this one with one button. Each device runs a small HTTP server reporting current playback state.
Channel Matcher (cross-provider) — identifies the same channel across different accounts despite different prefixes/suffixes, so you can zap to the same channel on a different provider when the primary fails.
VOD detection — DRM is explicitly NOT used as a VOD indicator (plenty of live channels use DRM too). VOD is detected via manifestType whitelist, group prefix (Movies:/Filme:/Series:/Serien:/VOD:) or URL extension.
In-app update dialog — version diff, in-app download and install.
Autostart wizard — step-by-step battery-whitelist, full-screen-notification permission, manufacturer-specific autostart settings deep-links.
Open-source-licensed dependencies listed in the about screen: AndroidX, Media3, Material, OkHttp, Coil, NanoHTTPD, Bouncy Castle, jcifs-ng, FFmpeg, WireGuard-for-Android, Font Awesome, XZ for Java.
