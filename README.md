# fuji-geotag

Backfill GPS coordinates into Fujifilm `RAF` / `HIF` / `JPG` files from a GPX
track, **without losing MakerNotes**.

Fujifilm RAF files store proprietary MakerNotes (Film Simulation, Lens info,
Dynamic Range, custom settings, etc.) in a block that many EXIF writers
inadvertently destroy when rewriting the file. This script uses the one
exiftool incantation that has been verified to leave the MakerNote block byte-
identical while only adding GPS tags.

## Why

Fujifilm bodies (X-T5, X-T4, X-H2, X100V, etc.) don't have built-in GPS. You
either pair the camera with the Fujifilm XApp/Cam Remote app over Bluetooth
and hope the link stays up, or shoot without GPS and backfill from a phone
track later.

If you've ever tried that "later" and ended up with all your film-simulation
metadata stripped, this script is for you.

## What's special

```bash
exiftool -overwrite_original_in_place -P -m \
  -geotag=<gpx> -geosync=<tz> -api GeoMaxExtSecs=<n> <files>
```

The critical flag is `-overwrite_original_in_place`. The default exiftool
write mode creates a new file and copies tags over, which can subtly
restructure the RAF/HEIF container and drop MakerNotes. In-place mode patches
the existing bytes for the affected tags only.

The script also:

- Backs up every candidate file before touching it (`/tmp/fuji-geotag-backup-<ts>/`).
- Verifies post-write by counting MakerNote tags in the modified file against
  the backup. If the count changes the script exits non-zero.
- Filters by date and by missing-GPS so re-runs are idempotent and cheap.

## Requirements

- `exiftool` 12+ (tested on 13.x). `brew install exiftool`.
- `bash` 4+ (the one shipped with macOS is 3.x; either install via brew or
  the script will work because it only uses `bash` 3-compatible features —
  except `mapfile`, which is 4+. **TODO**: replace `mapfile` with a portable
  loop if 3.x support matters.)

## Run without installing

```bash
curl -fsSL https://raw.githubusercontent.com/Innei/fuji-geotag/main/fuji-geotag \
  | bash -s -- --gpx ~/Downloads/track.gpx --dir /Volumes/Camera/DCIM/100_FUJI \
                --date 2026-06-01 --dry-run
```

Everything after `--` is passed to the script as if you'd installed it. Good
for one-shot use; review the source first if you want to be paranoid.

## Install

```bash
curl -fsSL -o ~/.local/bin/fuji-geotag \
  https://raw.githubusercontent.com/Innei/fuji-geotag/main/fuji-geotag
chmod +x ~/.local/bin/fuji-geotag
```

Or clone and symlink:

```bash
git clone https://github.com/Innei/fuji-geotag.git
ln -s "$PWD/fuji-geotag/fuji-geotag" ~/.local/bin/fuji-geotag
```

## Usage

```text
fuji-geotag --gpx <file.gpx> [--dir <photo-dir>] [options]
```

Required:

| Flag | Description |
|---|---|
| `--gpx <file>` | GPX track file |

Common:

| Flag | Default | Description |
|---|---|---|
| `--dir <path>` | `.` | Photo directory |
| `--date <YYYY-MM-DD>` | all | Filter by `DateTimeOriginal` prefix (e.g. `2026-06-01` or `2026-06`) |
| `--tz <±H:MM>` | `-9:00` | Camera-local clock vs. GPX UTC. JST is `-9:00`, China Standard `-8:00`, CET `-1:00`, etc. |
| `--max-ext <secs>` | `1800` | GPX extrapolation tolerance |
| `--exts <CSV>` | `RAF,HIF,JPG` | File extensions to consider |
| `-r, --recursive` | off | Descend into subdirectories |
| `--include-with-gps` | off | Overwrite files that already have GPS |

Safety:

| Flag | Description |
|---|---|
| `-n, --dry-run` | List candidates, don't write |
| `--no-backup` | Skip backup copy |
| `--backup-dir <path>` | Custom backup directory |
| `-v, --verbose` | Print every candidate |
| `-h, --help` | Show full help |

## Examples

Preview which photos in yesterday's shoot would be touched:

```bash
fuji-geotag --gpx ~/Downloads/track.gpx \
  --dir /Volumes/Camera/DCIM/100_FUJI \
  --date 2026-06-01 --dry-run
```

Actually write them:

```bash
fuji-geotag --gpx ~/Downloads/track.gpx \
  --dir /Volumes/Camera/DCIM/100_FUJI \
  --date 2026-06-01
```

A whole month:

```bash
fuji-geotag --gpx ~/Downloads/2026-track.gpx \
  --dir /Volumes/Camera/DCIM/100_FUJI \
  --date 2026-06
```

In China (UTC+8):

```bash
fuji-geotag --gpx ~/Downloads/track.gpx --tz -8:00 --dir .
```

Re-tag using a higher-quality GPX (overwrite existing GPS too):

```bash
fuji-geotag --gpx better.gpx --include-with-gps
```

## Manually setting a fixed coordinate

For photos with no track coverage where you know the location, the script
doesn't help — use exiftool directly with the same safe flags:

```bash
exiftool -overwrite_original_in_place -P -m \
  -GPSLatitude=35.7109 -GPSLatitudeRef=N \
  -GPSLongitude=139.7959 -GPSLongitudeRef=E \
  DSCF6135.RAF DSCF6135.HIF
```

## How it works (more detail)

1. **Discover.** Calls `exiftool -if 'not $GPSLatitude'` (plus optional date
   filter) over the directory to get the list of files that actually need GPS.
   This makes re-runs cheap and idempotent.
2. **Backup.** `cp -n` each candidate to a timestamped backup directory.
3. **Write.** A single `exiftool` invocation in `-overwrite_original_in_place`
   mode with `-P` (preserve mtime), `-m` (tolerate the harmless "Unknown N
   bytes at end of file" warning some HIFs trigger), `-geotag=<gpx>`,
   `-geosync=<tz>`, and `-api GeoMaxExtSecs=<n>`.
4. **Verify.** Counts files still missing GPS (should be 0). Picks one file,
   counts MakerNote tags in the modified copy vs. its backup, exits non-zero
   if the count differs.

The `-geosync` direction matters. Fujifilm bodies store `DateTimeOriginal` in
local time with no timezone tag. GPX is UTC. `-geosync=-9:00` tells exiftool
"subtract 9 hours from the camera clock when matching GPX points", which is
correct for JST. Use `-8:00` for China, `+0:00` if your camera is set to UTC.

`GeoMaxExtSecs` controls how far outside the GPX time range exiftool will
extrapolate. The default of 30 minutes covers the common case where you
forgot to start the tracker app for a few minutes; widen it for sparser
tracks.

## Tested with

- Fujifilm X-T5 RAF + HIF (HEIF lossless 10-bit)
- exiftool 13.x
- macOS 15 (darktable-cli is unrelated; only listed because it was used
  alongside this for color grading on the same shoot).

## License

MIT
