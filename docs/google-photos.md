# Bring your Google Photos library into Immich

⚠️ **Not yet tested end to end.** The steps follow the [immich-go](https://github.com/simulot/immich-go) documentation (version 0.32, the "Immich V3 compatibility" release). Reports welcome.

🇮🇹 [Italiano](it/google-photos.md)

## Why Takeout + immich-go

Google Takeout exports dates, places and descriptions in a separate `.json` file next to each photo, with names that don't always match (cut at 47 characters, `(1)`, `-edited`). The files themselves often carry the export date. Uploaded as they are, thousands of photos end up dated on the day of the export.

immich-go reads the Takeout zips directly (no need to extract them), matches each photo to its JSON, restores date, place and description, recreates your albums, and skips anything already in Immich (checksum match), so it can be re-run without creating duplicates.

## If your phone already syncs to the NAS

With the [external library](../README.md#3-add-the-external-library) set up, most of your recent photos are already in Immich. The Takeout is for the gaps: older phones, photos removed from the phone with "Free up space", photos saved from shared albums. The dry run in step 3 tells you how many are really missing. Photos whose metadata Google rewrote won't match by checksum and show up as duplicates: clean them up with **Utilities → Duplicates** in Immich.

## 1. Request the Takeout

Before requesting it, save to your own library the photos from albums **shared by others** that you want to keep: they are not yours for Takeout. Move out anything in the *Locked Folder* too.

On <https://takeout.google.com>:

1. **Deselect all**, then tick only **Google Photos** (keep all albums included).
2. Format **.zip**, maximum size **50 GB**, for fewer parts.
3. Google sends an email when it is ready (hours or days). Download **all** parts (`takeout-…-001.zip`, `-002.zip`, …) to your computer and check none is missing or truncated. The links expire after about a week.

## 2. Prepare an Immich API key

In Immich: *Account settings → API keys → New API key*, with these permissions:

`asset.read`, `asset.statistics`, `asset.update`, `asset.upload`, `asset.copy`, `asset.delete`, `asset.download`, `album.create`, `album.read`, `albumAsset.create`, `server.about`, `stack.create`, `tag.asset`, `tag.create`, `user.read`, `job.create`, `job.read`.

Keep the key in a password manager, never in a repo or a chat.

## 3. Dry run, then the real import

Install immich-go on your computer ([installation](https://github.com/simulot/immich-go/blob/main/docs/installation.md)) and run it at home, on the LAN (through Tailscale works too, just slower). Windows PowerShell:

```powershell
# dry run: reads everything and prints a report, uploads nothing
immich-go upload from-google-photos --dry-run `
  --server=http://<NAS LAN IP>:2283 --api-key=<API key> `
  --include-unmatched --include-partner=false --concurrent-tasks=4 `
  D:\Takeout\takeout-*.zip

# real import: the same command without --dry-run
```

| Option | Default | Why |
|---|---|---|
| `--sync-albums` | on | recreates your Google Photos albums |
| `--include-archived` | on | imports archived photos too |
| `--include-unmatched` | **off** | imports files without a JSON. **Turn it on**, or some files are skipped |
| `--include-partner` | on | imports photos shared by a partner. Set `false` if you don't want them in your library |
| `--include-trashed` | off | the trash stays out |
| `--people-tag` | on | Google's people names become tags (Google doesn't export face boxes; Immich recomputes faces) |
| `--pause-immich-jobs` | on | pauses thumbnails and ML during the upload |
| `--concurrent-tasks` | CPU cores | 4 is enough without choking the NAS |

At the end immich-go prints counters (found, uploaded, already present, missing JSON, errors) and writes a log with every file.

## 4. Check nothing is missing

1. **Counters:** images and videos found in the Takeout must equal uploaded + already present. Read errors and discarded files one by one in the log.
2. **Second pass:** run the same command again. It must say **0 uploaded**, everything already present.
3. **By year:** compare a few years in the Immich timeline with Google Photos (last year, an old year, a trip with many videos). Check dates of the oldest photos and of WhatsApp images and scans.
4. **Albums:** compare the album list and the item count of 3–4 large albums.

## What Takeout doesn't bring

- Photos in albums shared by others that you didn't save to your library.
- The *Locked Folder*, not always included.
- Full quality, if Google Photos was set to "Storage saver": you get the compressed copies, because the originals no longer exist on Google.
- Edits: the original and the `-edited` version arrive as two separate photos.
- Faces and search: Immich recomputes them; names must be assigned again (people tags help).

Pixel motion photos (`.MP.jpg`) are supported and stay animated.
