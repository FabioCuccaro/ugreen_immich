# Immich on a UGREEN NAS (UGOS, Docker)

Run [Immich](https://immich.app), a self-hosted Google Photos (timeline, albums, map, faces, search by content), on your UGREEN NAS. Immich reads the photos that the **UGREEN app already syncs** from your phone, without copying them, and you reach it from anywhere through **Tailscale**, without exposing anything to the internet.

🇮🇹 **Italiano:** [guida passo passo](docs/it/guida.md) · [foto da Google Photos](docs/it/google-photos.md)

## Who this is for

- You have a UGREEN NAS running **UGOS** (Pro) and you manage containers from the web UI, with **Docker → Project**. No SSH needed.
- Your phone already backs up its camera roll to the NAS with the UGREEN app, and you want a proper photo app on top of it.
- Optionally, you want to move your whole Google Photos library to the NAS.

## How it works

```
 phone (UGREEN app) ──sync──► /home/<user>/Photos/MobileBackup ──read-only──► Immich (external library)
 phone / laptop ──(Tailscale)──► http://nas:2283
```

- The official Immich compose (four containers: server, machine learning, Postgres, Valkey), adapted so it can be pasted into a UGOS Project without a separate `.env` file.
- The UGREEN sync folder is mounted **read-only** as an Immich *external library*. Immich indexes it in place: no second copy, no extra space, and the UGREEN sync keeps working as before.
- The Immich app's own backup stays **off**, otherwise every photo lands on the NAS twice.
- From outside, Immich is reached only through your tailnet (see [ugreen_tailscale](https://github.com/FabioCuccaro/ugreen_tailscale)). No Quick access, no UGREENlink, no open ports.

## Requirements

- **RAM:** Immich asks for at least 6 GB and recommends 8 GB ([requirements](https://docs.immich.app/install/requirements)). A DXP2800 (Intel N100, 8 GB) runs it next to Plex and a few other containers.
- **CPU:** the first indexing (thumbnails, metadata, faces, smart search, video encoding) keeps the NAS busy for hours to days, depending on library size. After that, each new photo takes seconds.
- **Database disk:** Postgres must be on a local disk of the NAS, never on a network share. An NVMe SSD is better; on HDDs it works, just slower (keep `DB_STORAGE_TYPE: 'HDD'` in the compose).
- **Tailscale** on the NAS, for access from outside: [ugreen_tailscale](https://github.com/FabioCuccaro/ugreen_tailscale).

## Setup

### 1. Find the path of your photos folder

The UGREEN app syncs the phone into your personal folder, typically shown in UGOS **Files** as *Personal folder → Photos → MobileBackup*. Inside Docker the same folder is `/home/<user>/Photos/MobileBackup`, where `<user>` is your UGOS user name.

To check: in **Files**, right-click the folder → **Properties**, or pick it with the folder selector of the Project editor.

### 2. Create the `immich` Project in UGOS

1. **Docker → Project → Create**. Name: `immich`. Folder: create `immich` inside your `docker` shared folder.
2. Paste [`docker-compose.yml`](docker-compose.yml) and replace:
   - `<database password>` with a random password made of **letters and digits only** (`A-Za-z0-9`). It is written once and reused for Postgres.
   - `/home/<user>/Photos/MobileBackup` with the path from step 1.
   - `TZ` with your timezone.
   - If `docker/immich` is on an SSD, delete the `DB_STORAGE_TYPE: 'HDD'` line.
3. Start it. The first start downloads several GB of images.
4. Open `http://<NAS LAN IP>:2283` and create the admin user.

Immich's own data goes to `docker/immich/library` (thumbnails, encoded videos, database dumps) and `docker/immich/postgres` (the database).

### 3. Add the external library

1. In the Immich web UI: your avatar (top right) → **Administration** → **External Libraries** → **Create library**, owner: your user.
2. In the new library: **Add path** → `/external/photos`. This is the path **inside** the container, not the NAS path.
3. When asked for a cron expression, that is the automatic rescan schedule: `0 * * * *` rescans every hour, `0 3 * * *` once a night. New photos from the UGREEN sync appear after the next scan.
4. **Scan**. Progress shows in **Administration → Jobs**, and the library page shows the photo and video counts.

If the count stays at 0: Docker → Container → `immich_server` → **Terminal**, run `ls /external/photos | head`. Empty output means the folder is not mounted; check the path in the compose.

### 4. Tune job concurrency (N100)

On a 4-core CPU with other containers running, high concurrency doesn't finish sooner, it just makes the whole NAS slow. In **Administration → Settings → Job settings**:

| Job | Concurrency |
|---|---|
| Video transcoding | 1 |
| Thumbnail generation | 2 |
| Metadata extraction | 2 |
| Smart search | 1 |
| Face detection | 1 |
| OCR (if present) | 1 |
| Library, Sidecar, Migration, Background tasks | defaults |

After the first indexing these values hardly matter. You can raise thumbnails and metadata to 3–4 at night; keep video and machine learning at 1. Any queue can be paused from the Jobs page while you watch Plex.

### 5. Phone app

1. Install the **Immich** app and sign in with `http://nas:2283` (through Tailscale).
2. **Do not turn on the app's backup.** The UGREEN app already uploads the photos. Phone photos will show a "not backed up" icon in the Immich app: that is expected, Immich sees them through the external library.
3. Optional: *Settings → Networking → Automatic URL switching*: your home Wi‑Fi with `http://<NAS LAN IP>:2283`, and `http://nas:2283` as the external URL.
4. On Android, make Tailscale an *Always-on VPN* so the app keeps working away from home.

### 6. Updates

The compose uses the `v3` tag: re-pulling the images (Project → update / redeploy in UGOS) gets every Immich 3.x release, never Immich 4. **Immich has no `:stable` tag**, and `release` would jump to the next major version unattended. When Immich 4 is out, read the release notes and change `v3` to `v4` by hand.

## Optional: bring over your whole Google Photos library

The UGREEN sync copies what is **on the phone**, not what is on Google Photos: photos from older phones, photos removed from the phone with "Free up space" and photos saved from shared albums are missing. Google Takeout plus [immich-go](https://github.com/simulot/immich-go) fills the gaps, with a `--dry-run` first to see how many are really missing. Guide: [`docs/google-photos.md`](docs/google-photos.md). ⚠️ Not yet tested end to end.

## Backups

⚠️ Not yet tested.

RAID protects against a broken disk, not against theft, fire, a wrong delete or a failed update. Until there is a copy outside the NAS, keep Google Photos (or another cloud backup) running in parallel.

What to copy, for example with the UGOS backup app to a USB disk or an encrypted cloud:

- the photos folder (`/home/<user>/Photos/MobileBackup`);
- `docker/immich/library`, especially `library/backups`, where Immich writes a nightly database dump ([backup and restore](https://docs.immich.app/administration/backup-and-restore)).

Don't copy `docker/immich/postgres` while the database is running: restore from the dump instead.

## Security notes

- **Never commit secrets**: the filled-in compose (database password), Immich API keys, the `library/` and `postgres/` folders. [`.gitignore`](.gitignore) excludes the usual paths.
- No Quick access / UGREENlink, no open ports on the router, no Tailscale Funnel for Immich: it holds your whole photo history.
- Port 2283 is plain HTTP. On your LAN that is the same as any other NAS service; from outside it travels inside Tailscale, encrypted.

## Tested on

| | |
|---|---|
| NAS | UGREEN DXP2800 (Intel N100, 8 GB), UGOS, Docker app → Project |
| Image | `immich-server:v3` (Immich 3.3, October 2026) |
| Install, external library on the UGREEN sync folder | ✅ |
| Android app through Tailscale | ✅ |
| Google Takeout + immich-go | ⚠️ untested |
| Backups | ⚠️ untested |

Tried it on another UGREEN model? Please open an issue or a PR to add it to this table.

## Contributing

Corrections and reports from other NAS models are welcome. UGOS and Immich menu names change between versions: if what you see on screen differs from this guide, the screen wins, and a PR fixing the guide helps the next person.

## License

[MIT](LICENSE). Not affiliated with UGREEN, Immich or Tailscale.
