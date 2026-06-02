---
name: fuji-geotag
description: Backfill GPS coordinates into camera files (RAW/HEIF/JPG) from a GPX track using exiftool, while preserving camera-vendor MakerNotes (Fujifilm Film Simulation, Sony/Nikon/Canon lens & shooting metadata, etc.). Use when the user wants to add or fix GPS in EXIF from a phone track, geotag a shoot, recover missing GPS, or when a previous geotag attempt destroyed Film Simulation / MakerNote data.
---

# fuji-geotag

A bash + exiftool wrapper that geotags photos safely.

Repository: https://github.com/Innei/fuji-geotag

## When to use

- User has photos missing GPS in EXIF and a GPX track from their phone.
- User has been burned by `exiftool -geotag` stripping Fujifilm Film Simulation or other MakerNote data.
- User wants to batch-backfill an entire day/month/shoot.
- User asks how to geotag RAW files (RAF / ARW / NEF / CR3) safely.

## When NOT to use

- User only wants to view location on a map — just open GPX in their DAM.
- User wants per-photo manual editing — recommend a GUI (HoudahGeo, exiftool GUI, etc.).
- User's camera already embeds GPS reliably.

## The key insight

The default `exiftool -geotag` rewrites the file, which can drop the MakerNote
block on Fujifilm RAF and some HEIF containers. The fix is one flag:

```bash
exiftool -overwrite_original_in_place -P -m \
  -geotag=<gpx> -geosync=<tz> -api GeoMaxExtSecs=<n> <files>
```

- `-overwrite_original_in_place` — patches existing bytes only, no rebuild
- `-P` — preserves file mtime
- `-m` — ignores harmless "Unknown N bytes at end of file" warnings
- `-geosync=<±H:MM>` — offset of camera local time vs GPX UTC
- `-api GeoMaxExtSecs=<n>` — extrapolation tolerance in seconds

## Running the script

One-shot, no install:

```bash
curl -fsSL https://raw.githubusercontent.com/Innei/fuji-geotag/main/fuji-geotag \
  | bash -s -- --gpx ~/Downloads/track.gpx --dir . --date 2026-06-01
```

Or install permanently:

```bash
curl -fsSL -o ~/.local/bin/fuji-geotag \
  https://raw.githubusercontent.com/Innei/fuji-geotag/main/fuji-geotag
chmod +x ~/.local/bin/fuji-geotag
```

Prereq: `brew install exiftool` (or apt/pacman equivalent).

## Common invocations

Dry-run to see what would be touched:

```bash
fuji-geotag --gpx ~/Downloads/track.gpx \
  --dir /Volumes/Camera/DCIM/100_FUJI \
  --date 2026-06-01 --dry-run
```

Backfill yesterday:

```bash
fuji-geotag --gpx ~/Downloads/track.gpx \
  --dir /Volumes/Camera/DCIM/100_FUJI \
  --date 2026-06-01
```

Whole month, custom timezone, non-Fuji extensions:

```bash
fuji-geotag --gpx track.gpx \
  --dir /shoots/2026-06 \
  --date 2026-06 \
  --tz -8:00 \
  --exts ARW,JPG \
  --recursive
```

## Parameters reference

| Flag | Default | Purpose |
|---|---|---|
| `--gpx <file>` | required | GPX track file |
| `--dir <path>` | `.` | Photo directory |
| `--date <YYYY-MM-DD>` | all | DateTimeOriginal prefix filter (`2026`, `2026-06`, `2026-06-01` all valid) |
| `--tz <±H:MM>` | `-9:00` | JST→UTC. Use `-8:00` for China, `+0:00` for UTC cameras |
| `--max-ext <secs>` | `1800` | GPX extrapolation tolerance (30 min default) |
| `--exts <CSV>` | `RAF,HIF,JPG` | File extensions |
| `-r, --recursive` | off | Descend into subdirs |
| `--include-with-gps` | off | Re-tag files that already have GPS |
| `-n, --dry-run` | off | List candidates, don't write |
| `--no-backup` | off | Skip backup (default backs up to `/tmp/fuji-geotag-backup-<ts>/`) |
| `--backup-dir <path>` | auto | Custom backup location |

## Fixed-coordinate fallback (when GPX has no coverage)

For known locations where the GPX has a gap, use exiftool directly with the
same safe flags:

```bash
exiftool -overwrite_original_in_place -P -m \
  -GPSLatitude=35.7109 -GPSLatitudeRef=N \
  -GPSLongitude=139.7959 -GPSLongitudeRef=E \
  DSCF6135.RAF DSCF6135.HIF
```

## Verification habit

After any geotag write, sanity-check that vendor metadata is intact:

```bash
diff <(exiftool -G -a -s -MakerNotes:all backup/DSCF6041.RAF) \
     <(exiftool -G -a -s -MakerNotes:all /Volumes/Camera/.../DSCF6041.RAF)
# Should print "Files are identical" or nothing.
```

The script does a tag-count check automatically and exits non-zero on
mismatch, but a manual diff is cheap insurance for important shoots.

## Pitfalls

- **Timezone direction.** `-geosync=-9:00` means "subtract 9h from camera
  clock to match GPX UTC", correct for cameras set to JST. If camera is set
  to UTC use `+0:00`; if camera is set to China time use `-8:00`. Wrong sign
  → all GPS points off by 18 hours.
- **No coverage = snap to boundary.** If shots predate the GPX track start
  by more than `GeoMaxExtSecs`, exiftool refuses; under it, exiftool snaps
  to the nearest boundary point. Widen `--max-ext` carefully — wider means
  more potential drift.
- **Already-tagged files are skipped by default.** Use `--include-with-gps`
  to re-tag from a new authoritative GPX.
- **SD-card writes are slow.** Copying to a local SSD before tagging, then
  back, can be much faster for big batches.
