# Build a self-hosted Gyazo replacement: ShareX → n8n → Immich

> A build guide for a Claude Code session (or any LLM harness) setting this up from scratch
> in a **new** environment. Derived from a working installation built 2026-09-19/20 against
> Immich 2.5.6 and n8n 2.4.6.
>
> Placeholders: `<IMMICH_HOST>`, `<N8N_HOST>`, `<TOKEN>`, `<IMMICH_KEY>`, `<ALBUM_ID>`.
> Read §2 and §8 before writing anything — they will save you the most time.

---

## 1. What you are building

Press a hotkey on a Windows machine → screenshot uploads to self-hosted Immich → filed into
a `Screenshots` album → archived out of the main photo timeline → the Immich deep link lands
on the clipboard. Screenshots become searchable by **OCR text**, by **CLIP object/scene
recognition**, and by **source application**.

```
[ShareX on client] --HTTPS/HTTP POST--> [n8n webhook] --REST--> [Immich API]
       ^                                      |                      |
       |______________ {"url": ...} __________|                      v
        (clipboard)                                      [OCR + CLIP indexing, async]
```

### Why a webhook and not a watched folder

The tempting simpler design is: ShareX saves into a network share, Immich's *external
library* scans it. **Don't.** Immich's scanner is periodic, so the asset does not exist at
the moment of capture, so there is no id and no URL to hand back to ShareX — you lose the
clipboard link, which is the entire Gyazo experience. The webhook responds *synchronously*
after the asset is filed, which is what makes the link possible.

A second reason: if Immich's own upload storage ever lands **inside** an external-library
import path, the scanner re-discovers files Immich already owns and you get duplicate
assets. Keep managed storage and scanned libraries strictly separate.

---

## 2. Decisions to make before building

**2.1 Archive the screenshots, or leave them in the timeline?**

Archiving keeps captures out of the main photo timeline. The cost: **archived assets are
excluded from default search results** — both text and semantic. Finding them requires
"Include archived" in the UI, or `"visibility":"archive"` via API.

- Large existing photo library → **archive**. Dozens of daily screenshots will otherwise
  bury your photos, and one search toggle is cheaper.
- Immich used mainly *for* screenshots → **don't archive**. Search works by default.

ML indexing is unaffected either way, and switching later needs no re-indexing — the OCR
text and embeddings already exist. This is a reversible, low-stakes decision.

**2.2 Album + archive, or a separate Immich user?** A separate user isolates screenshots
completely but complicates search across both. Album + archive is simpler; this guide uses it.

**2.3 Where do originals live?** Immich writes uploads under its `UPLOAD_LOCATION`. If that
is not where your backups look, see §7 — and read §8.3 before trying to relocate it with a
bind mount.

---

## 3. Prerequisites

- Immich running and reachable (note the exact version — API shapes drift; see §5)
- Immich machine-learning container running; OCR requires Immich **≥ 2.2**
- n8n running, with the **Webhook** and **HTTP Request** nodes (stock)
- Network path from n8n to Immich, and from each client to n8n
- ShareX on each client
- Optional but much faster: an n8n API key (Settings → n8n API) so credentials and the
  workflow can be created programmatically

**Verify n8n can actually reach Immich before building anything:**

```bash
docker exec <n8n_container> sh -c "wget -qO- http://<IMMICH_HOST>:2283/api/server/version"
```

If the containers are on different Docker networks, container-name DNS will **not** resolve
between them. Use the host's LAN IP rather than editing either compose file.

---

## 4. Build

### 4.1 Create the album

Create an album named `Screenshots` in the Immich **web UI**, then read its id:

```bash
curl -s -H "x-api-key: <IMMICH_KEY>" http://<IMMICH_HOST>:2283/api/albums
```

Do it in the UI because the runtime API key should *not* hold `album.create` — it needs to
create albums exactly once, ever.

### 4.2 Create the Immich API key with minimal scopes

Immich → Profile → API Keys. The runtime pipeline needs **only three**:

```
asset.upload        POST /api/assets
asset.update        PUT  /api/assets/<id>        (archive)
albumasset.create   PUT  /api/albums/<id>/assets
```

Grant extra scopes temporarily for verification, then remove them:
`asset.read` (inspect state), `asset.view` + `asset.download` (prove bytes round-trip),
`asset.delete` (clean up test assets), `album.read` (find the album id).

The plaintext is shown once; Immich stores it hashed and it cannot be recovered.

### 4.3 Generate a webhook token

```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

### 4.4 Create two n8n credentials

Both are type `httpHeaderAuth`. Via the n8n API:

```json
POST /api/v1/credentials
{"name":"ShareX Webhook Token","type":"httpHeaderAuth","data":{"name":"X-ShareX-Token","value":"<TOKEN>"}}
{"name":"Immich API Key","type":"httpHeaderAuth","data":{"name":"x-api-key","value":"<IMMICH_KEY>"}}
```

Record both ids. **n8n cannot read credentials back**, so save the values elsewhere now.
Never inline secrets in the workflow JSON — assert it before deploying:

```python
assert token not in json.dumps(workflow) and immich_key not in json.dumps(workflow)
```

### 4.5 Build the workflow

Seven nodes, linear with a shared error branch:

```
Webhook → Normalize → Upload to Immich → Add to Album → Archive → Respond OK
              ↘            ↘                 ↘            ↘
                            Respond Error (HTTP 500)
```

| Node | Type | Config |
|---|---|---|
| Webhook | `n8n-nodes-base.webhook` v2 | POST, path `sharex-capture`, `authentication: headerAuth` (token credential), `responseMode: responseNode` |
| Normalize | `n8n-nodes-base.code` v2 | `runOnceForEachItem`; code below |
| Upload | `httpRequest` v4.2 | POST `http://<IMMICH_HOST>:2283/api/assets`, `contentType: multipart-form-data`, Immich credential |
| Add to album | `httpRequest` v4.2 | PUT `…/api/albums/<ALBUM_ID>/assets`, jsonBody `={{ JSON.stringify({ids:[$json.id]}) }}` |
| Archive | `httpRequest` v4.2 | PUT `…/api/assets/{{ $('Upload to Immich').item.json.id }}`, jsonBody `={{ JSON.stringify({visibility:'archive'}) }}` |
| Respond OK | `respondToWebhook` v1.1 | json: `={{ JSON.stringify({url:'http://<IMMICH_HOST>:2283/photos/' + $('Upload to Immich').item.json.id}) }}` |
| Respond Error | `respondToWebhook` v1.1 | text, `options.responseCode: 500` |

Set `onError: "continueErrorOutput"` on Normalize, Upload, Add-to-album and Archive, and
wire each node's **output index 1** to Respond Error. Without this a failure returns a
misleading success or hangs.

Upload body parameters:

```
assetData        parameterType: formBinaryData, inputDataFieldName: "file"
deviceAssetId    ={{ $json.deviceAssetId }}
deviceId         ={{ $json.device }}
fileCreatedAt    ={{ $json.capturedAt }}
fileModifiedAt   ={{ $json.capturedAt }}
isFavorite       false
```

Normalize code — accepts whatever binary field name the client actually sends, and derives
a per-machine tag from an optional header:

```javascript
const binary = $input.item.binary || {};
const keys = Object.keys(binary);
if (keys.length === 0) {
  throw new Error('no file in request: expected a multipart file field');
}
const src = binary[keys[0]];
const fileName = src.fileName || 'capture.png';
// ShareX uploads immediately on capture, so upload time == capture time to within seconds.
const capturedAt = new Date().toISOString();
const hdrs = $input.item.json.headers || {};
const rawDevice = (hdrs['x-sharex-device'] || '').toString().trim();
const device = (rawDevice || 'unspecified').toLowerCase().replace(/[^a-z0-9._-]/g, '-').slice(0, 64);
return {
  json: {
    fileName,
    capturedAt,
    device: `sharex-${device}`,
    deviceAssetId: `sharex-${device}-${Date.now()}-${fileName}`,
  },
  binary: { file: src },
};
```

Deploy with `POST /api/v1/workflows`, then `POST /api/v1/workflows/<id>/activate`.

### 4.6 Configure each client

ShareX custom uploader (`.sxcu` — double-click to import, or write `UploadersConfig.json`
directly if the `Version` field mismatches your installed ShareX):

```json
{
  "Version": "17.0.0",
  "Name": "Immich",
  "DestinationType": "ImageUploader",
  "RequestMethod": "POST",
  "RequestURL": "http://<N8N_HOST>:5678/webhook/sharex-capture",
  "Headers": {
    "X-ShareX-Token": "<TOKEN>",
    "X-ShareX-Device": "<machine-name>"
  },
  "Body": "MultipartFormData",
  "FileFormName": "file",
  "URL": "{json:url}"
}
```

Then, in ShareX:

1. Destinations → Image uploader → **Custom image uploader**, and select this uploader.
2. Task Settings → After capture → **tick "Upload image to host"**. *This is the most
   commonly missed step* — without it ShareX saves locally and never contacts the webhook,
   with no error explaining why.
3. Task Settings → After upload → tick **"Copy URL to clipboard"**.
4. File naming → set a pattern including **window title and process name** (e.g. the
   equivalent of `%pn - %t-%y%mo%d_%h%mi%s`). Read exact tokens from ShareX's own variable
   picker. The default is timestamp-only, which permanently destroys app identification —
   the filename becomes Immich's `originalFileName`, and screenshots carry no EXIF, so it is
   the *only* channel for "which app was this from".
5. Turn **off** `ImageAutoUseJPEG`. It silently re-encodes large captures to lossy JPEG,
   degrading exactly the text OCR depends on.

Give each machine a distinct `X-ShareX-Device`. One shared token means revoking it revokes
every machine; for per-machine revocation, create one credential and webhook auth per client.

---

## 5. Verify the API shapes against *your* Immich — do not trust this document

Immich's REST API drifts between versions, and **Swagger may not exist** (on 2.5.6,
`/api/docs`, `/api/docs/json`, `/api/specs`, `/api/openapi.json` are all 404). Probe the
live API with your key and confirm each call before wiring it.

Shapes confirmed on **2.5.6**:

| Operation | Call | Response |
|---|---|---|
| Upload | `POST /api/assets` multipart | `201 {"id":…,"status":"created"\|"duplicate"}` |
| Add to album | `PUT /api/albums/<id>/assets` `{"ids":[…]}` | `200 [{"id":…,"success":true}]` |
| Archive | `PUT /api/assets/<id>` `{"visibility":"archive"}` | `200` |
| Deep link | `/photos/<assetId>` | SPA route |

Shapes that **used to** work or are commonly assumed and are **wrong** on 2.5.6:

- `PUT`/`POST /api/archives` with `{assetIds, isArchived}` → **404**. Archive is now a
  `visibility` field on the asset.
- `PUT /api/albums/<id>/assets` with `{"assetIds":[…]}` → **400** `"ids must be an array"`.
- `"withArchived": true` in search → silently ignored. Use `"visibility":"archive"`.

Determine the deep-link route from the web build rather than probing HTTP — an SPA returns
`200` for *every* path, so probing tells you nothing:

```bash
docker exec <immich_server> sh -c 'grep -rhoE ".{70}\"/photos.{70}" /build/www/_app/immutable | head -3'
# 2.5.6 yields:  viewAsset:({id:r})=>`/photos/${r}`
```

---

## 6. Verification checklist

Do all of it with real output. *Green-by-assertion does not count.*

1. `curl -X POST <webhook> -H "X-ShareX-Token: <TOKEN>" -F "file=@fresh.png"` → `200` + `{"url":…}`
2. Same without the header → `401`/`403`
3. Same **with** the token but no file → `500` + short error (exercises the error branch)
4. `GET /api/assets/<id>` → `visibility`, `isArchived`, `deviceId`, `originalFileName` all correct
5. `GET /api/albums/<ALBUM_ID>` → contains the asset
6. `GET /api/assets/<id>/original` → sha256 matches the file you sent
7. OCR: upload an image containing a distinctive nonsense string, then
   `POST /api/search/metadata {"ocr":"<string>","visibility":"archive"}` → returns it
8. A real hotkey capture, with the link landing on the clipboard and opening the asset

Two traps that make tests lie:

- **Immich deduplicates by checksum.** Re-uploading identical bytes returns the *existing*
  asset id, so a repeated test looks like it passed while nothing happened. Generate a
  distinct image every time (e.g. random fill colour).
- **Semantic search always returns ~100 ranked candidates** — it ranks, it never filters. In
  a small album *every* query "matches". To prove object recognition genuinely works, rank
  your test asset against the **whole** library and confirm unrelated queries do *not*
  surface it. A control query is mandatory here.

---

## 7. Backup

**Images alone are close to useless.** Files on disk are UUID-named; the original filename,
album membership, archive state, OCR text and embeddings all live in **Postgres**. Any
restorable backup must include the database.

Immich already writes nightly Postgres dumps into `<UPLOAD_LOCATION>/backups/`. Back up:

| Folder | Keep? |
|---|---|
| `upload/` | **yes** — the originals, irreplaceable |
| `backups/` | **yes** — the DB dumps that make the originals meaningful |
| `profile/`, `library/` | yes, tiny |
| `thumbs/` | no — regenerable |
| `encoded-video/` | no — regenerable |

Skipping the last two is the difference between ~3 GB and ~150 GB.

An rsync mirror, scheduled an hour after Immich's dump, onto a **different physical disk**:

```bash
#!/bin/bash
set -uo pipefail
SRC=<upload_location>; DST=<backup_target>
# refuse to mirror a vanished source over a good backup
[ -d "$DST" ] || { echo "ABORT: destination missing"; exit 1; }
[ -f "$SRC/upload/.immich" ] || { echo "ABORT: source marker missing"; exit 1; }
rsync -a --delete \
  --include='upload/***' --include='backups/***' \
  --include='profile/***' --include='library/***' --exclude='*' \
  "$SRC/" "$DST/"
```

Those two guards matter: `--delete` makes this a mirror, and a mirror of an unmounted
source *erases your backup*. Steady state is flat (Immich rotates its own dumps), not
growing — but retention is then capped at Immich's rotation window, so keep periodic
snapshots separately if you want deeper history.

**Restore:** load the newest dump into a fresh Immich's Postgres, copy `upload/` back, let
Immich regenerate thumbnails and video.

---

## 8. Pitfalls, in the order they will bite you

**8.1 "Upload image to host" unticked.** Client-side, silent, and looks like a server fault.
Check it first whenever no request arrives.

**8.2 Archive hides screenshots from search.** Expect to explain "Include archived" to every
user. See §2.1.

**8.3 Never bind-mount a subdirectory *inside* an existing bind mount** to relocate Immich's
originals. On Docker Desktop/WSL2 with a Windows-drive (9p/drvfs) source, this **hangs
container start indefinitely** — the container sits in `Created`, produces no logs, `docker
start` times out, and the daemon wedges badly enough that only restarting Docker Desktop
clears it. It cost ~45 minutes of downtime on the reference build.

If originals must move, relocate `UPLOAD_LOCATION` wholesale (and migrate the existing data),
or leave storage alone and back it up elsewhere as in §7.

**Always test a mount spec on a throwaway container first:**

```bash
docker run --rm -v <src>:<dst> alpine true   # hangs in seconds if the spec is bad
```

Component checks — folder writable, marker readable, path resolvable — can all pass
individually while the actual combination fails. Test the combination.

**8.4 Immich's `.immich` mount markers can block startup.** Immich records per-folder
`mountChecks` flags in Postgres. Once flagged, it will **not** recreate a missing marker: it
throws `ImmichStartupError` and refuses to start. If you bind-mount over any of `upload`,
`thumbs`, `library`, `profile`, `backups`, `encoded-video`, pre-create a `.immich` file in
the new directory (contents arbitrary). `IMMICH_IGNORE_MOUNT_CHECK_ERRORS=true` is the
escape hatch.

**8.5 Docker Desktop resolves Linux-style paths differently per client context.** A
container created from the *Windows* CLI may resolve `/mnt/d/...` to an **empty
auto-created directory**, while the same spec from inside a WSL distro resolves to the real
drive. Symptom: a recreated container silently loses access to a library mount. Resolution
is fixed at container **creation**, so restarts are safe — only recreation re-resolves.
Test with a throwaway container before recreating anything that matters.

**8.6 Different Docker networks.** n8n and Immich in separate compose projects cannot
resolve each other by container name. Use the host IP.

**8.7 Tooling.** In Git Bash, `export MSYS_NO_PATHCONV=1` before passing Unix paths to
`docker`/`wsl`. Avoid deep nested quoting (Windows shell → wsl → bash → docker) — variables
get silently eaten and produce confidently wrong output; write a script file instead. `curl
-F` breaks on filenames with spaces or colons. Piping masks exit codes.

**8.8 Fullscreen captures carry no app name.** No active window ⇒ no `%pn`/`%t` ⇒
timestamp-only filename. Unfixable at any layer. They remain findable by OCR and CLIP.

**8.9 Client config may be cloud-synced.** ShareX's `UploadersConfig.json` holds the token
in plaintext. If ShareX's personal folder sits under OneDrive/Dropbox, the token syncs to
that provider. Relocate the folder or accept it knowingly.

**8.10 Check whether your n8n is publicly exposed.** If `WEBHOOK_URL`/`N8N_HOST` point at a
tunnel (ngrok, Cloudflare), the webhook is not LAN-only and the token is genuinely
internet-facing. Inbound path matching ignores `WEBHOOK_URL`, so a LAN URL still works.

---

## 9. What you get when it works

- Hotkey → link on clipboard in well under a second (177–531 ms observed)
- Screenshots out of the main timeline, collected in one album
- Search by **OCR text** (verified: rendered text read back exactly), by **object/scene**
  (verified: a clipped cat photo ranked 4th of ~41,000 assets, while unrelated queries did
  not surface it at all), and by **source app** via filename
- Per-machine attribution via `deviceId`
- Failures return `500` with a reason, never a link to a half-filed asset
