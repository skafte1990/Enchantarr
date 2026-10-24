# Enchantarr

![Enchantarr](https://raw.githubusercontent.com/skafte1990/Enchantarr/main/assets/enchantarr-banner.png)

Official releases and installation guide. This repository contains documentation and downloads only, no source code.

Enchantarr watches your Radarr and Sonarr queues and, before a download is imported into
your library, produces a corrected copy of the file: audio/subtitle track flags
(default/forced/disabled, by language, codec, channel count, and title keywords) are
set, and video/audio streams are conditionally transcoded (only if they don't already
match your configured target codec) — then hands that copy to Radarr/Sonarr's Manual
Import API. The original downloaded file is never modified, so actively-seeded
torrents are never at risk.

Replaces `mkvpriority`.

## Running

Quick start (Docker Compose):

```yaml
services:
  enchantarr:
    image: shaaft/enchantarr:latest
    container_name: enchantarr
    restart: unless-stopped
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Europe/Copenhagen
    ports:
      - "8000:8000"
    volumes:
      - /path/to/enchantarr/data:/data
      - /path/to/downloads:/downloads
      - /path/to/movies:/movies
      - /path/to/tv:/tv
    #devices:
    #  - /dev/dri:/dev/dri
    #deploy:
    #  resources:
    #    reservations:
    #      devices:
    #        - driver: nvidia
    #          count: 1
    #          capabilities: [gpu]
```

Environment variables:

- `PUID` / `PGID` — user/group ID the container runs as (linuxserver.io convention),
  so files it creates in `/data` and your mounted download/library folders have the
  right ownership on the host. Default `1000`/`1000` if omitted.
- `TZ` — container timezone, e.g. `Europe/Copenhagen`. Optional.
- `ENCHANTARR_SECRET_KEY` — random string used to sign session cookies. If omitted, one
  is generated automatically and persisted to `/data/secret_key` on first boot.
- `ENCHANTARR_STAGING_DIR` — where Enchantarr writes its working copy of a file while
  processing it. Defaults to `/downloads/.trimarr-staging`. Only change this if your
  shared downloads mount inside the container isn't `/downloads` — see below for why.

(Older `TRIMARR_*` names for these variables still work.)

Mount `/data` for the SQLite database and app secret (Enchantarr-private, doesn't need to
be visible to anything else).

Mount your Radarr/Sonarr downloads folder read/write at **the same container path
Radarr and Sonarr themselves use for it** (`/downloads` in the example compose,
matching the usual trash-guides/servarr convention of every download-adjacent
container mounting the same host folder at the same internal path). This isn't
optional: Enchantarr both reads the original file from that path (it comes straight from
Radarr/Sonarr's own queue data) *and* writes its processed staging copy there, then
hands that staging file's path to Radarr/Sonarr's Manual Import API. If Radarr/Sonarr
don't have that exact path mounted too, they won't be able to find the staged file and
the import will fail. Nothing needs to be mounted for Enchantarr's staging copies
specifically — they live in a hidden subfolder of the same downloads mount
(`.trimarr-staging`) and are deleted again once each job finishes, successfully or not.

**Before-import mode needs Radarr/Sonarr to never import anything on their own** —
Enchantarr's own Manual Import call must be the only way a file ever gets imported, or
Radarr/Sonarr can race Enchantarr and import the untouched original first (confirmed
live, twice: Completed Download Handling's own completed-download scan is typically
faster than Enchantarr's queue poll interval, and separately, a download client's own
"Remove Completed Downloads" setting removes a finished download from Radarr/Sonarr's
queue before Enchantarr's next poll ever sees it — even with Completed Download Handling
off — so the file is silently never processed by anything at all). Enchantarr manages
both of these automatically: switching to before-import under **Settings → General**
and hitting Save turns Completed Download Handling off and Remove Completed Downloads
off on every configured connection's download client(s). If Enchantarr can't reach a
connection when you save, it still saves its own setting and shows you which
connection needs fixing manually (Settings → Download Clients in Radarr/Sonarr
itself) instead of failing silently. (After-import mode doesn't have this requirement
— it deliberately lets Radarr/Sonarr import normally first, then processes the
library copy afterward, and switching to it turns Completed Download Handling back
on.)

After starting the container, open `http://<host>:8000` — you'll land straight on the
Dashboard, no login required. Add your Radarr and Sonarr connections under
**Connections**, and build a scoring/encoding **Flow** for each one (guided setup, no
raw config files). Active transcode jobs show live progress under **Jobs**.

### Login / authentication

Enchantarr has no login by default, matching Radarr/Sonarr's own out-of-the-box
behavior — anyone who can reach the web UI can use it, same as an unconfigured
Radarr/Sonarr instance. To require a login, go to **Security** and choose one of:

- **Disabled** (default) — no login, ever.
- **Disabled for local addresses** — login is only required for requests that
  don't come from a private/LAN IP (e.g. someone reaching Enchantarr from outside your
  network through a reverse proxy still needs to log in; devices on your own LAN
  don't).
- **Enabled** — login always required.

The first time you turn on either non-disabled mode, you'll be asked for a username
and password on that same screen — Enchantarr logs you in with those credentials
immediately after saving, so you won't get bounced out mid-setup.

**Locked out?** Set `ENCHANTARR_RESET_AUTH=true` in the container's environment and
restart it once — this resets authentication back to **Disabled** (it does not touch
your username/password) so you can get back in and review your Security settings.
Remove the variable again afterward, or it'll reset authentication on every restart.

### Hardware video encoding

Optional. If you pass through a GPU to the container (see the commented-out sections
in `docker-compose.example.yml`), set a Flow's encoder backend to "hardware" or "auto"
and pick the matching encoder from the dropdown (NVENC/QSV/VAAPI/AMF). Without GPU
passthrough, use "software".

## Trailarr connection

[Trailarr](https://github.com/nandyalu/trailarr) downloads trailers into a `trailers/`
subfolder inside each movie or series folder. A Trailarr connection lets Enchantarr
transcode those trailers using the same Flow the matching Radarr or Sonarr connection
already uses, so crop detection, Dolby Vision handling, language handling, and your
hardware encoder settings all apply to trailers too. The trailer file is replaced in
place; Radarr and Sonarr are never touched for it.

**Requirements.** You must already have a Radarr and/or Sonarr connection configured.
That connection's library is how Enchantarr matches a downloaded trailer file back to the
media folder it belongs to.

**Adding it.** Go to **Settings → Connections → Add connection** and pick connection
type "Trailarr". The Trailarr URL and API key are optional but recommended: with them
Enchantarr resolves Trailarr's own media id to the exact Radarr/Sonarr item; without them
it matches on title and year against your library, which is less precise. The Trailarr
API key is in Trailarr's own settings.

**Wiring the webhook.** After adding the connection, copy the webhook URL shown on its
card. In Trailarr, go to **Settings → Notifications → Add channel** and paste that URL
as the Apprise URL, prefixed with `json://` (or `jsons://` if your Enchantarr is served
over HTTPS). Tick only the "Trailer Downloaded" event, leave "Include User Events"
off, and use the Test button to confirm it reaches Enchantarr. Example:

    json://192.168.1.50:8587/api/webhook/trailarr/<secret>

If you regenerate the webhook secret on the connection card, the old URL stops working
and you have to paste the new one into Trailarr.

**Do you need it?** Trailarr can already convert trailers itself through its Trailer
Profiles, so if you only want a consistent codec that may be enough on its own. Use a
Trailarr connection when you want your full Enchantarr Flow, with its Dolby Vision strip,
crop detection, language handling, and encoder settings, applied to trailers as well.

## Enchantarr Node (Windows, optional)

A Windows app that runs transcodes on another machine, so a stronger PC or GPU can take the load.

1. In Enchantarr, open **Nodes**, add an external node and copy its connection string
   (`http://<host>:<port>?token=...`).
2. Download the latest installer (`enchantarr-node-setup-<version>.exe`) from the [Releases page](https://github.com/skafte1990/Enchantarr/releases/latest) and run it.
3. Click the tray icon, open **Settings** and paste the connection string.
4. In Enchantarr, open the node and set its **path mapping** so that each shared folder
   (for example `/downloads`, `/movies`) points to the same folder as the node sees it
   (for example `\\nas\media\downloads`). The node needs read and write access to those folders.

The tray panel shows what the node is doing. When a new version is available, press **Update**
in the panel; the node downloads it from your Enchantarr server and restarts by itself.
