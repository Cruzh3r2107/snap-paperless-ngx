# snap-paperless-ngx

Paperless-ngx is a community-supported open-source document management system that transforms your physical documents into a searchable online archive so you can keep, well, less paper.

This repository packages it as the `vsingh-paperless` snap, built from upstream
`paperless-ngx/paperless-ngx` at tag `v3.0.5`. Redis, the OCR toolchain
(Tesseract, Ghostscript, ImageMagick, OCRmyPDF) and the Angular web UI are all
bundled — there is nothing external to install or run.

## Install

```bash
sudo snap install --dangerous ./vsingh-paperless_3.0.5_amd64.snap
```

## Set up

Create your login before you can use the web UI:

```bash
sudo snap set vsingh-paperless admin-user=admin admin-password=changeme
```

Then browse to <http://localhost:8000>.

## Configuration

```bash
sudo snap set vsingh-paperless port=8000
sudo snap set vsingh-paperless url=https://paperless.example.com
sudo snap set vsingh-paperless ocr-language=eng
sudo snap set vsingh-paperless time-zone=Europe/London
```

Changing any of these rewrites the config and restarts the services for you.

For anything not exposed as a snap option, edit
`/var/snap/vsingh-paperless/common/paperless.conf` directly and run
`sudo snap restart vsingh-paperless`. That file is seeded on first install with
a generated secret key and is **never overwritten on refresh**.

Bundled OCR languages are English, German, French, Italian and Spanish.

## Your data

Everything lives under `/var/snap/vsingh-paperless/common/`:

```
data/         SQLite database and the search index
media/        your archived documents
consume/      drop files here and they are ingested automatically
export/       document exports
paperless.conf
```

This is `$SNAP_COMMON`, so it survives refreshes and reverts. Back up `media/`
and `data/` together — the archive is worth little without its database.

## Interfaces

`home`, `network`, `network-bind` and `shared-memory` connect automatically.

To consume or export from mounted drives and network shares under `/media` or
`/mnt`, connect this one by hand:

```bash
sudo snap connect vsingh-paperless:removable-media
```

## Services

Six apps: `webserver`, `worker`, `scheduler`, `consumer` and `redis` are
daemons; `manage` is the Django CLI.

```bash
snap services vsingh-paperless
snap logs vsingh-paperless -f
sudo vsingh-paperless.manage check
```

## Troubleshooting

The web server binds all interfaces on port 8000, so it is reachable from your
LAN. Put it behind a reverse proxy or a VPN if that is not what you want.

Lossy JBIG2 PDF/A compression is unavailable: `jbig2enc` is not in the Ubuntu
archive and is not bundled. OCRmyPDF simply skips that step — OCR itself is
unaffected.

---

Build instructions and packaging internals are in
[SNAP_PACKAGING.md](SNAP_PACKAGING.md).
