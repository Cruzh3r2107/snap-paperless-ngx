# Packaging paperless-ngx as a snap

This snap packages [paperless-ngx](https://github.com/paperless-ngx/paperless-ngx)
v3.0.5 as a strictly-confined, self-contained document management service for
`core24` (Ubuntu 24.04). Redis is **bundled** — no external broker is required.

## What is inside

- **Multi-part build** (dual language):
  - `frontend` — Node 24 (via the `node/24/stable` build-snap) + pnpm/corepack
    builds the Angular UI into `src/documents/static/frontend/`.
  - `backend` — Python 3.12 dependencies installed with `uv` (torch from the
    PyTorch CPU index, `--index-strategy unsafe-best-match`), the `src/` tree,
    NLTK data, `collectstatic` + `compilemessages`, and the ImageMagick PDF
    policy.
  - `wrappers` — the `bin/paperless-*` launcher scripts.
- **Bundled OCR/processing toolchain** via `stage-packages`: tesseract (eng,
  deu, fra, ita, spa), ghostscript, imagemagick, unpaper, pngquant, jbig2dec,
  qpdf, poppler-utils, gnupg, icc-profiles-free, libmagic, etc.
- **Bundled Redis** (`redis-server`) run as an internal daemon on TCP loopback (127.0.0.1:6379)
  with its data under `$SNAP_COMMON/redis`. Bound to loopback only, so it is
  never reachable off the host.
- **SQLite by default** at `$SNAP_COMMON/data/db.sqlite3`. PostgreSQL support is
  compiled in (`--extra postgres`) but not active unless configured.

## Services (apps)

| App          | Type        | Command                              | Plugs |
|--------------|-------------|--------------------------------------|-------|
| `webserver`  | daemon      | Granian ASGI, port 8000              | network, network-bind |
| `worker`     | daemon      | Celery worker                        | network, home, removable-media |
| `scheduler`  | daemon      | Celery beat                          | network |
| `consumer`   | daemon      | `manage.py document_consumer`        | network, home, removable-media |
| `redis`      | daemon      | bundled Redis (127.0.0.1:6379)          | — |
| `manage`     | CLI         | `manage.py <cmd>`                    | network, home, removable-media |

## Prerequisites

```bash
sudo snap install snapcraft --classic
sudo snap install lxd
sudo lxd init --auto
```

## Build

```bash
cd snap-paperless-ngx
export SNAPCRAFT_BUILD_INFO=1
snapcraft pack
```

> This is a **large** build (torch CPU wheel is ~2 GB; the ML stack adds more).
> Expect a multi-GB snap and a long build. Running the `snap-trimmer` skill
> afterwards is recommended.

## Install & test

First test in devmode to shake out AppArmor issues:

```bash
sudo snap install --devmode ./vsingh-paperless_3.0.5_amd64.snap
```

Then test real strict confinement:

```bash
sudo snap install --dangerous ./vsingh-paperless_3.0.5_amd64.snap
```

Browse to <http://localhost:8000>.

### Create the admin user

```bash
sudo snap set vsingh-paperless admin-user=admin admin-password=changeme
```

Other runtime options handled by the `configure` hook:

```bash
sudo snap set vsingh-paperless port=8000
sudo snap set vsingh-paperless url=https://paperless.example.com
sudo snap set vsingh-paperless ocr-language=eng
sudo snap set vsingh-paperless time-zone=Europe/London
```

## Interface connections

These auto-connect on a classic desktop but must be connected manually on
Ubuntu Server / Core (and always for `--dangerous` installs of unassertioned
snaps):

Auto-connecting (`network`, `network-bind`, `home`) — usually no action needed.

Manual (**not** auto-connected):

```bash
# Consume/export from mounted external drives or network shares (/media, /mnt)
sudo snap connect vsingh-paperless:removable-media
```

If `home` is not auto-connected in your environment:

```bash
sudo snap connect vsingh-paperless:home
```

## Configuration

`$SNAP_COMMON/paperless.conf` is seeded on first install (with a generated
`PAPERLESS_SECRET_KEY`) and **never overwritten on refresh**. It redirects all
paths into `$SNAP_COMMON`:

- `PAPERLESS_DATA_DIR=$SNAP_COMMON/data`
- `PAPERLESS_MEDIA_ROOT=$SNAP_COMMON/media`
- `PAPERLESS_CONSUMPTION_DIR=$SNAP_COMMON/consume`
- `PAPERLESS_EXPORT_DIR=$SNAP_COMMON/export`
- `PAPERLESS_LOGGING_DIR=$SNAP_COMMON/log`
- `PAPERLESS_CONVERT_TMPDIR=$SNAP_COMMON/tmp`
- `PAPERLESS_NLTK_DIR=$SNAP/usr/share/nltk_data`
- `PAPERLESS_STATICDIR=$SNAP/static`
- `PAPERLESS_REDIS=redis://localhost:6379`

Edit it directly and restart services:

```bash
sudo snap restart vsingh-paperless
```

## Troubleshooting

- Inspect a confined shell: `sudo snap run --shell vsingh-paperless.manage`
- Watch AppArmor/SecComp denials: `sudo journalctl -xe | grep -i denied`
- Service logs: `sudo snap logs vsingh-paperless -f`
- Verify Redis is up: `sudo snap run --shell vsingh-paperless.manage -c 'redis-cli -h 127.0.0.1 ping'`

## Notes / caveats (from the analysis)

- **Redis is mandatory** and bundled as an internal loopback-TCP daemon. It
  plugs `network` and `network-bind` because AppArmor socket-address mediation
  denies an AF_UNIX pathname `bind()` under strict confinement, so a Unix socket
  is not an option — see the comment in `snap/local/bin/paperless-redis`.
  `--bind 127.0.0.1` is what keeps it off the network, not the absence of plugs.
- **jbig2enc is NOT in the Ubuntu archive.** Upstream installs a prebuilt `.deb`
  from `paperless-ngx/builder`. It is **omitted** here (only the `jbig2dec`
  decoder is staged). OCRmyPDF simply skips lossy JBIG2 PDF/A compression when
  the encoder is absent — OCR itself is unaffected.
- **torch (~2 GB CPU wheel)** plus scikit-learn / sentence-transformers /
  llama-index make this a very large snap. Consider `snap-trimmer`.
- **PostgreSQL**: the `postgres` extra is installed (per the analysis) but the
  upstream `psycopg-c` prebuilt wheel targets Debian trixie glibc; if you switch
  to PostgreSQL and hit a load error, install `libpq5` and re-verify. SQLite
  (the default) needs none of this.
- **flower** monitoring UI was intentionally omitted to keep the service set
  lean; add it as an extra `network-bind` daemon if desired.
- **Migrations & search index**: run by the `install` and `post-refresh` hooks,
  and (best-effort) at webserver start. `document_index reindex --if-needed`
  runs once Redis is up.
