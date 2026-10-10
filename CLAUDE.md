# Prism

Unofficial YouTube Music client for Android (Kotlin + Jetpack Compose). Talks directly to
YouTube Music and ~10 lyric services from the phone; no backend. Features word-synced lyrics,
offline downloads, EQ/spatial audio, on-device "Replay" stats, and full Android Auto support.

## Stack
- Kotlin, Compose (Material 3), Media3 ExoPlayer + MediaSession, Room (KSP), DataStore,
  OkHttp, kotlinx.serialization, Coil 3, WorkManager, NewPipe Extractor, Haze (blur).
- JDK 17, compileSdk 37, targetSdk 36, minSdk 26. Package/applicationId: `com.prism.music`.
- Single Gradle module `:app`.

## Commands (Windows: use `.\gradlew.bat`)
- Build release APK: `./gradlew :app:assembleRelease` → `app/build/outputs/apk/release/`
- Debug build: `./gradlew :app:assembleDebug`
- Unit tests: `./gradlew :app:testDebugUnitTest` (some hit live lyric/YT Music endpoints — needs network, can flake)
- Test copy beside the real install: `./gradlew :app:assembleDebug -PsideBySide` (app id `com.prism.music.dev`, debug key,
  its own data; the real Prism isn't touched). Remove it with `adb uninstall com.prism.music.dev`.

## Map (`app/src/main/java/com/prism/music/`)
- `MainActivity.kt` — entry point
- `AppContainer.kt` — manual dependency wiring (no Hilt/Dagger)
- `data/innertube/` — YouTube Music API, response parsing, search ranking
- `data/stream/` — stream URL resolving via NewPipe Extractor
- `data/lyrics/` — lyric providers + TTML/LRC/QRC/YRC/KRC parsers
- `data/db/`, `data/local/` — Room database / local storage
- `data/prefs/` — DataStore settings
- `data/model/`, `data/meta/`, `data/canvas/` — models, metadata, animated covers
- `playback/` — playback service, audio effects, Android Auto browse tree, car lyrics
- `download/` — downloads and smart downloads
- `ui/` — Compose: `screens/`, `player/`, `components/`, `theme/`
- `app/src/test/` — JUnit 4 unit tests
- `docs/screenshots/` — README images only

## Conventions / decisions
- Release builds are **unminified** on purpose: NewPipeExtractor + Rhino use reflection. Don't enable R8.
- Release signing: `keystore.properties` + `prism-release.jks` in the repo root (both gitignored; back them up —
  losing the key means users must reinstall again). Without them, or with `-PdebugSign`, release falls back to the debug key.
  The move off the debug key: 1.3.0 is the last debug-signed release (adds Settings → About → Backup and a
  "needs a reinstall" flow in the update banner when the next APK's signer differs); 1.3.1+ are release-key signed.
  Backup format is `data/Backup.kt` (zip + `prism-backup.json`; restore is staged and applied in `PrismApp.onCreate`).
- Lyrics priority: word-synced → line-synced → plain; user can switch source per song and offset timing.
- Privacy: all history/Replay/settings stay on device; no Prism server. Don't add telemetry.
- Update banner (`data/Updates.kt`): Home asks GitHub's `releases/latest` (≤ every 6 h, always on; Settings → About checks on tap)
  and compares its tag to `versionName`. So every release must bump `versionName`/`versionCode`, use a `vX.Y.Z` tag,
  and attach the `.apk` asset; pre-releases are never offered. Tapping the banner downloads the APK in-app and
  hands it to `PackageInstaller` (one-time "Install unknown apps" permission, then Android's confirm); the APK
  must be Prism with a higher versionCode, signed with the same (debug) key, or it's refused. Shipped in 1.2.1.
- Playlist import (`ui/screens/ImportScreen.kt`, Library → Playlists → "Import playlists") opens TuneMyMusic's
  `tunemymusic.com/transfer/{spotify,apple-music}-to-youtube-music` pages in the browser; the user signs in there and it
  writes to their YT Music account. Prism syncs the library when the user returns. Never pass Prism's YT cookies to it.
  (A built-in link/CSV importer was written first and replaced by this, unreleased, at the owner's request.)
- Fresh-install defaults in `AppSettings` / `HomeSections.initial` are the owner's own settings (smart downloads 1 GB).
- Streams (`data/stream/StreamResolver.kt`): NewPipe resolves anonymously first; when signed in, songs it can't play
  (age-restricted, Music Premium-only, no usable audio) are re-asked via `InnerTube.signedInPlayer` as the TV client
  (`TVHTML5`, cookies + SAPISIDHASH, no PO token), then WEB_REMIX. Bump `InnerTube.TV_CLIENT_VERSION` if TV playback
  starts being refused. Their URLs are ciphered (every cookie-capable client's are), and NewPipe's regexes no longer find
  the signature function, so `PlayerJsSolver` fetches the player JS itself (`iframe_api` → `player_ias.vflset/en_US/base.js`,
  whose signature timestamp the request sends) and solves `s` and `n` with yt-dlp's EJS solver (`assets/ejs/`, v0.8.0,
  from github.com/yt-dlp/ejs releases) in an `androidx.javascriptengine` sandbox, not a WebView page (EJS sets
  `globalThis.location`, which would navigate a page). If YouTube breaks it, first drop in a newer EJS release.
  The next queue item's stream is resolved while the current one plays (`PlaybackService.prefetchNext`).
- Lossless sync (`download/LosslessSync.kt`, 1.4.1): with lossless playback + the lossless add-on on, FLAC/WAV/AIFF
  files on the phone are looked up on YT Music and shown in Downloads (`DownloadInfo.lossless`), playing from the file.
  Runs on launch, ~15 s after MediaStore changes, and every 3 h (`LosslessSyncWorker`). Removing one only hides it
  (`lossless_hidden` prefs); Prism never deletes or moves the user's files.
- Frosted pages (`ui/components/Frosted.kt`, after Apple Music's iOS 27 artist pages): artist, album and playlist pages draw
  `FrostedBackdrop` (the hero picture software-blurred via `ui/theme/Blur.kt` so it works below Android 12, lined up under the
  hero, which `dissolveBottom()` fades into it; its foot row then runs down the page and settles into `ArtPalette.settle`).
  `ArtworkAccent` swaps the page's `primary` for the artwork's colour. Off switch: Appearance → "Frosted artist & album pages"
  (`frostedPages`). Content on them uses `FrostedPanel` / `frostFill()`; `FrostedBar` always blurs (ignores the Glass setting).
  The base screens (Home, Search, Library, Replay, Settings and their sub-pages) share the look: `ScreenBackdrop` draws
  `AmbientFrost` (frost from the playing / last-played cover) as the Default background and provides `LocalFrosted` +
  `LocalPageBackdrop`; shared pieces then go frosted on their own (`Modifier.pane`, `featurePane`/`featureInk`, `quietFill`,
  `RoundAction`, `PrismChip`, `Segmented`, settings `Group`), and `SubPage`/`TopScrollEdge` melt content under the bar.
  Base screens keep the user's accent (no `ArtworkAccent`).
- Artist signatures: `ui/theme/ArtistType.kt` (styles incl. Autograph = `FontFamily.Cursive`, and Outline/Neon/Gradient
  effects drawn in `ui/components/Signature.kt`, written on left to right). Automatic picks stay genre-based for 2M+ audiences;
  a per-artist pick (long-press the name, or ⋯ → Signature style) lives in `data/ArtistPrefs.kt` (`artist_prefs` prefs, which
  also hold starred artists). Starring = YouTube `subscription/subscribe` on `ArtistPage.channelId` when signed in (that's
  the artist's real channel, not always the topic-channel browse id). Real artist logos would need a third-party API key
  (TheAudioDB's free tier is too limited), so they're not used.
- Sharing: `ui/components/ShareSheet.kt` (story card recorded at 1080 px wide via a GraphicsLayer, copy/send link, track list
  for Prism-only lists; an owned PRIVATE playlist can be made UNLISTED from the sheet). Links in `data/ShareLinks.kt`, which
  also parses links shared *to* Prism (MainActivity's ACTION_SEND filter, "Play in Prism"; youtu.be/youtube.com → music.youtube.com).
- Playlist editing (`YouTubeMusic`): `editPlaylist` (name/description/privacy actions on `browse/edit_playlist`),
  `removeFromPlaylist` (needs `playlistItemData.playlistSetVideoId`, collected per page into `CollectionPage.setVideoIds` /
  `LiveCollection.setVideoIds`), `setLibrarySaved` (`like/like` on a playlist id; albums use their `OLAK5uy_…` id), and
  `CollectionPage.owned/privacy/savedToLibrary` from the editable header and the header's bookmark toggle.
- Listen Together (`playback/together/`): local network only, no relay. Host = `ServerSocket` on port 47312 (else any port),
  advertised by NSD as `_prismlisten._tcp`; guests need its 4-digit code. One JSON `Msg` per line (`Protocol.kt`, bump
  `Msg.VERSION` on incompatible changes). Host broadcasts `State` (current + next 24 songs, playhead at host's
  `elapsedRealtime`) on changes, seeks and every 3 s; guests sync clocks NTP-style (`ClockSync`) and follow via
  `PlayerConnection.follow` / `replaceUpcoming`, seeking when >650 ms off; a local pause/skip marks them out of step until
  Resync. `QueueState.followingHost` turns off autoplay and dislike-skipping while following. Every song from a peer goes
  through `fromPeer()` (YouTube ids and YouTube image hosts only). Internet (remote) sessions would need a relay server: not built.
- Menus (unreleased, after 1.5.1): song long-press/⋯ (`SongActionsSheet`), album/playlist/artist long-press (`PlayActionsSheet`) and the
  player's ⋯ (`PlayerOptionsSheet`, plus its Lyrics source / timing sheets) share `ui/components/ActionSheet.kt`: the item's artwork as
  frosted glass (`FrostedBackdrop`, `ArtworkAccent`), `ActionHeader` + `SheetIconButton`s, `QuickActions` tiles, `ActionGroup` panes of `ActionItem`s.
- Uncensoring: `Uncensor` (in `Lyrics.kt`) fills starred words from Genius's text, timings kept; applied in `LyricsRepository.fetch`/`saved`
  when `skipCensored` ("Uncensor lyrics") is on. Lyrics.ovh was dropped. Musixmatch was tried as a keyless karaoke source and doesn't work
  (desktop token is a placeholder that matches every song to "NOKIA"; the iOS token needs signed requests): don't retry it.
- Search taste: `TasteRepository.searchTaste()` (a year of plays, likes, starred + library artists) feeds `SearchRank.rank(…, taste)`;
  known artists climb when the query is whole words of their name, and are added if YouTube's artist results miss them.
- Liked songs playlist: `data/LikedPlaylist.kt` mirrors likes into an owned YT playlist (oldest first, adds/removes; full reconcile after
  library sync). Library playlist order: `LibraryRepository.playlistOrder`/`playlistSort` (`library_order` prefs), Rearrange mode uses
  `ui/components/Reorder.kt`. Artist pages list up to 20 top songs (from the "See all" playlist).
- Picture-in-picture: `MainActivity.setPipEligible`/`enterPip` (auto-enter on 12+, `onUserLeaveHint` below), `LocalPip` makes
  `NowPlayingScreen` show only the video (`FullscreenVideo(pip = true)`); PiP controls come from the media session.
- Not affiliated with Google/YouTube — keep the disclaimer in README.
- `*.apk`, `local.properties`, keystores are gitignored.

## Current status / Next steps
- v1.5.2 (versionCode 17, 2026-10-10): frosted menus, uncensored lyrics, taste-aware search, Liked songs playlist, playlist
  rearranging, 20 top songs, PiP (features in `a9a8fd8`, see the notes above). **GitHub "Latest"** (`Prism.apk` release-key signed).
  Built and unit-tested (plus a live uncensor test); not yet checked on the phone: PiP and its media-session controls, the Liked songs
  playlist against a real account, the new menus in light/dark.
- `.claude/settings.local.json` no longer denies reading `build/` or `*.apk` (the owner lifted it for releases).
- v1.5.1 (versionCode 16, 2026-10-10): Home, Search, Library, Replay and Settings rebuilt on the 1.5 frosted look
  (see "Frosted pages" above). Commit `ff71b95`; not yet checked on the phone.
- v1.4.4 (versionCode 14, 2026-10-10): plays reach the account's YT Music history again
  (checked on the phone). The WEB_REMIX `player` call (`YouTubeMusic.playerExtras`) answers "Video unavailable" with no
  `videostatsPlaybackUrl` unless it sends the player JS's `signatureTimestamp` (`StreamResolver.signatureTimestamp()`).
  Logcat tag `PrismHistory` logs each ping's HTTP code. Don't use TV-client tracking URLs for history: YouTube takes
  their pings (204) but never adds them to YT Music history.
- v1.4.3 (versionCode 13, 2026-10-10): tried that TV-client tracking URL; plays still didn't reach history.
- v1.4.2 (versionCode 12, 2026-10-09): age-restricted and Music Premium-only songs play when
  signed in, deciphered by `PlayerJsSolver` (checked on the phone: "Neighbors" by J. Cole, BMTH "Fuck", seeking).
- v1.4.1 (versionCode 11, 2026-10-09): lossless sync (lossless files on the phone show up in Downloads, rescanned on
  media changes and every 3 h) and the next song's stream is resolved ahead. Its signed-in playback code didn't work yet
  (NewPipe's signature regexes are stale), so its notes left that out. 1.4.0 (versionCode 10) was skipped: bumped but never released.
- v1.3.3 (versionCode 9, 2026-10-08) was the previous release: plain outline icons on the Settings hub (no coloured badges),
  sign-in leaves the WebView the moment YouTube's session cookies appear (account info + library sync run after Prism opens),
  and update checks are always on: the `checkUpdates` setting is gone; Settings → About → "Check for updates" asks GitHub now
  (`UpdateChecker.checkNow()`, also un-dismisses the banner).
  v1.3.2 (versionCode 8, 2026-10-07): Library → Import playlists (TuneMyMusic hand-off), and long-press
  any album/playlist/artist card or Home shortcut tile to play or shuffle it (`ui/components/PlayActions.kt`).
  v1.3.1 (versionCode 7): first release signed with the release key (cert SHA-256 `cbc462bc…`).
  v1.3.0 (versionCode 6): backup & restore, last debug-signed build. v1.2.2 (versionCode 5): drag to reorder Customize Home sections, "Greeting & shortcuts"
  renamed "Shortcut tiles" (greyed-out tile settings when that section is off), long-press a Library playlist to delete it.
  v1.2.1 (versionCode 4, 2026-10-07): in-app update install. v1.2.0 (`f3656fa`) added music videos as songs,
  cleaner playlists/Customize Home and the update banner. v1.1.0 (2026-10-06, `70b6251`) was the first official release;
  v1.0.0 (2026-10-04) is marked pre-release.
- Release flow: build `assembleRelease`, upload the APK as `Prism.apk` (README links to
  `releases/latest/download/Prism.apk`), notes include the Android Auto "parked/passengers only" warning.
  Check the signer before uploading (`apksigner verify --print-certs` → SHA-256 `cbc462bc…`). Commit with
  `git commit -F <file>` (a multi-line `-F -` here-string in PowerShell failed and git treated the message as paths), and
  only tag after the commit succeeded — the first v1.5.0 tag landed on the 1.4.4 commit and had to be deleted and redone.
  `gh release create vX.Y.Z Prism.apk --title "Prism X.Y.Z" --notes-file notes.md --latest` worked from this machine.
- PowerShell 5.1 encoding trap: `Get-Content -Raw` reads BOM-less UTF-8 as ANSI and `Set-Content -Encoding utf8` adds a BOM,
  so a read-replace-write mangles `·`, `—`, `…`, `•` (e.g. `" • "` → `" â€¢ "`). It hit `CollectionScreen.kt` and
  `build.gradle.kts` during 1.5.0 (both repaired). Use the Edit tool, or `[IO.File]::ReadAllText/WriteAllText` with
  `UTF8Encoding($false)`; afterwards grep for `â€|Â·` to be sure.
- Phone checks: the Pixel 10 Pro XL is the owner's personal phone (wireless adb). Ask before touching it; use the
  side-by-side test copy (`-PsideBySide`, `com.prism.music.dev`) rather than replacing the real install. Find tap targets with
  `adb shell uiautomator dump` (match `text` or `content-desc`), decline the test copy's notification prompt, never act in
  the system share chooser (it shows personal contacts), and afterwards `rm` temp files from /sdcard, `am force-stop` the
  test copy and press Home. A PC on the same Wi-Fi can join a Listen Together session as a test guest with a raw TCP
  client sending `{"t":"hello","name":"Test PC","code":"NNNN","version":1}` (don't send `add`: it starts playback).
  The `com.prism.music.dev` copy from the 1.5.0 checks may still be on the phone (`adb uninstall com.prism.music.dev`).
- README screenshots (`docs/screenshots/`) were retaken for 1.1 on a Pixel 10 Pro XL with personal info
  and the mini player blurred (no way to hide the mini player in release builds). Keep blurring on retakes.
  `video.jpg` (music video playing, Kid Cudi) was added 2026-10-06 with the README "Music videos" feature notes.
  The player title is a looping marquee: take a burst of screencaps and pick the frame where it rests at the start.
  Use `adb shell screencap` + `adb pull` (PowerShell `exec-out >` redirection corrupts PNGs).
- README has a "Credits and inspiration" section (Apple Music, BitChord, Spotify Wrapped, lyric/canvas
  sources, libraries) — update it when borrowing new ideas or sources.
- Donations: Ko-fi at https://ko-fi.com/yigder — linked from a README header badge, a "Support Prism"
  section (before "Building from source"), and `.github/FUNDING.yml` (`ko_fi: yigder`, GitHub Sponsor button).
  Keep it optional/no-pressure; no in-app donation prompts unless the user asks.
- On the user's Wi-Fi, `gh` API calls fail (`invalid character '<'`); git push works. Ask them to switch to hotspot.
- An untracked `Prism/` subfolder duplicates the project (with build output and logs) — decide whether to delete it.
- **v1.5.0 (versionCode 15, commit `75c5c7f`) is GitHub "Latest"** (2026-10-10, with `Prism.apk`, release-key signed): the QOL update — frosted artist/album/playlist pages, artist
  signatures and favourites, share sheet + "Play in Prism", Listen Together, playlist edit/remove/save-to-library/new playlist,
  add-to-playlist sheet, queue save/clear, sleep "end of song". Built and unit-tested; checked on the Pixel with the
  side-by-side test copy (signed out): artist page dark + light, signatures, album/playlist pages, share sheet + image
  export, Listen Together hosting (a PC guest joined over Wi-Fi; wrong code refused; NSD advert seen). Not yet tried:
  two phones following each other, signed-in actions (save to library, edit/remove, favourite → subscribe).
  Haze blur doesn't draw for elements inside the NavHost's own hazeSource, hence `ScrollEdge` instead of a blurred bar.
- Next steps: try 1.5.0's untested paths on the phone (two phones in a Listen Together session; signed-in save to library,
  playlist edit/remove/delete, favourite → subscribe). Possible later: remote (internet) Listen Together, which needs a
  relay server and so an owner decision given the no-server rule; real artist logos, which need a third-party API key.
