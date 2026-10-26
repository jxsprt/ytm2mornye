# ytm2mornye

> **Status: blueprint.** This repository contains the design and the plan only. There is no implementation yet — see [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for the full design and the build roadmap.

`ytm2mornye` turns a **YouTube Music playlist** into a **local music library** by handing its tracks to [Mornye](https://github.com/mornyeapp/Mornye-Python), the Python music-library tool that matches tracks and retrieves audio through the provider extensions *you* install and configure.

```
music.youtube.com/playlist?list=PL…  ──►  ytm2mornye  ──►  mornye --csv  ──►  ~/Music/<playlist>/*.flac + .m3u8
```

## Why this exists

Mornye's core resolves Spotify links natively and accepts a **CSV of tracks** anywhere it accepts a link. YouTube Music links only work in Mornye if a YouTube provider extension is installed, and that path downloads YouTube's own (lossy) audio.

`ytm2mornye` fills the gap:

1. It reads the playlist from YouTube Music (via [`ytmusicapi`](https://github.com/sigma67/ytmusicapi)).
2. It normalizes every entry into clean `title / artist / album / duration` rows, dropping or flagging things that are not really songs (user uploads, podcast episodes, unavailable videos).
3. It writes a Mornye-compatible CSV and runs Mornye on it. Mornye matches each row against its catalogue, rejects weak matches instead of guessing, and downloads through your configured providers.

You end up with your YouTube Music playlist as a tagged local library at the quality your providers offer, plus an M3U8 playlist file, and a report of every track that could not be matched.

## Planned features

- **Playlist sources:** public playlists, album "playlists" (`OLAK5uy_…`), and, if you authenticate, private playlists and *Liked Music* (`LM`).
- **Two handoff modes:**
  - `match` (default): text rows go through Mornye's catalogue matching and your lossless providers.
  - `direct`: passes `music.youtube.com` links straight to Mornye. This only works if you have installed a YouTube provider extension, and the audio is lossy.
- **Song-only filtering:** classifies entries by YouTube Music video type (official audio, music video, user upload, episode) so a fan upload or a 1-hour mix doesn't end up in your library unnoticed.
- **Incremental sync:** a re-run only downloads what is new. Removed tracks are reported and never deleted automatically.
- **Unmatched-track report:** a CSV of misses that you can edit and feed back in.
- **Dry run:** shows exactly what would be matched and downloaded, with no audio fetched.
- **Watch mode:** keeps a folder in sync with a playlist on an interval.
- **Machine-readable output** (`--json`) for scripting and notifications.

## Planned usage

> These commands describe the target interface. They don't work yet.

```bash
# One-time Mornye setup: point it at a registry you trust and install a provider
export MORNYE_REGISTRIES="https://example.com/your-registry.json"

# See what would happen (no downloads)
ytm2mornye plan "https://music.youtube.com/playlist?list=PLxxxxxxxx"

# Sync a playlist into ~/Music
ytm2mornye sync "https://music.youtube.com/playlist?list=PLxxxxxxxx" --dest ~/Music

# Several playlists into one library, shared tracks downloaded once
ytm2mornye sync PLaaaa PLbbbb --dest ~/Music --quality HI_RES_LOSSLESS

# Only export the CSV (inspect it, or run Mornye yourself)
ytm2mornye export PLxxxxxxxx -o roadtrip.csv
mornye --csv roadtrip.csv ~/Music --csv-unresolved missed.csv

# Private playlists / Liked Music (browser-header auth from ytmusicapi)
ytm2mornye auth setup          # writes ~/.config/ytm2mornye/browser.json
ytm2mornye sync LM --dest ~/Music

# Keep it in sync every 6 hours
ytm2mornye watch PLxxxxxxxx --dest ~/Music --every 360
```

Planned configuration file (`~/.config/ytm2mornye/config.toml`):

```toml
dest = "~/Music"
mode = "match"                 # match | direct
include = ["song", "official_video"]   # song | official_video | user_upload | episode
min_score = 0.62               # forwarded to mornye --csv-min-score

[mornye]
services = ["ext:tidal-web", "ext:qobuz-web"]
quality = "LOSSLESS"
filename_format = "{artist} - {title}"
m3u = "m3u8"
```

## Requirements (planned)

| Requirement | Why |
| --- | --- |
| Python ≥ 3.11 | Matches Mornye's floor |
| [`mornye`](https://pypi.org/project/mornye/) | Matching, downloading, tagging, M3U output |
| [`ytmusicapi`](https://pypi.org/project/ytmusicapi/) | Reading YouTube Music playlists |
| At least one Mornye provider extension | Mornye ships with **no** providers and **no** default registry. You choose and install them. |
| `ffmpeg` (optional) | Only needed if you use Mornye's `transcode_to` |

## Repository layout

```
.
├── README.md              ← you are here
├── LICENSE                ← MIT
└── docs/
    └── ARCHITECTURE.md    ← components, data model, sequence, decisions, roadmap
```

The planned source layout (`src/ytm2mornye/…`) is described in [ARCHITECTURE.md § Planned module layout](docs/ARCHITECTURE.md#7-planned-module-layout). It will be added when implementation starts.

## Responsible use

This tool is meant for **personal use** to build a local copy of music you are entitled to access. It hosts, bundles, and distributes **no** audio, and like Mornye it ships **no** download provider.

You are responsible for following the laws where you live and the Terms of Service of YouTube Music and of any provider you configure in Mornye. `ytm2mornye` is not affiliated with Google/YouTube or with the Mornye project.

## License

[MIT](LICENSE)
