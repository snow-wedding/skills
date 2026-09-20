# Plaud → Local-Transcription → Notion Pipeline — Agent Setup Guide

**Audience:** an AI agent (Hermes, Claude Code, or equivalent) performing the setup on a new machine.
**Goal:** replicate this workflow: Plaud Note recordings are polled and downloaded, transcribed + diarized locally (zero Plaud AI credits), summarized on a local/private LLM, rendered to an HTML report, and published as toggles on per-day Notion pages — on a 15-minute cron with manual on-demand triggers.

You (the agent) are expected to run gate checks yourself, ask the human only for what you cannot obtain, and delegate coding tasks to a coding agent (e.g. `claude-code` CLI) where indicated.

---

## Phase 0 — What you're building

```
Plaud Note Pro (device syncs over Wi-Fi)
   │
   ├─ plaud-mcp (npx @plaud-ai/mcp, stdio MCP)
   │     list_files + get_file ONLY → presigned S3 audio URL
   │     ⚠ NEVER call get_transcript / get_note — they spend the user's
   │       paid Plaud transcription/summary credits. That is the point of
   │       this pipeline: replace Plaud's AI with local AI.
   │
   ├─ WhisperX (local GPU)  large-v3 + pyannote diarization → transcript.json
   ├─ vLLM / LM Studio (local)  schema-strict JSON summary → summary.json
   ├─ render: inject merged data into meeting-report-template.html
   └─ Notion (API): find-or-create day page → toggle per recording
```

State machine: SQLite `state.db`, one row per (recording, stage); every stage is idempotent (skips when outputs are newer than inputs). Cron tick = discover → process stages → publish. An overlap lock (`.tick.lock`) makes concurrent ticks exit silently.

Choose a project root (below: `$ROOT`, e.g. `C:\pipeline` on Windows or `~/plaud-pipeline` on Linux). Create `scripts/`, `recordings/`, `logs/`, `prompts/`, `docs/` under it.

---

## Phase 1 — Gate checks (run these yourself before asking for anything)

| # | Check | Command | Pass condition |
|---|---|---|---|
| G1 | Node ≥ 20 (plaud-mcp requires it) | `node --version` | v20+ |
| G2 | NVIDIA GPU + recent driver | `nvidia-smi` | GPU visible, driver supports CUDA 12.8 (Blackwell/RTX 50xx needs torch cu128; older GPUs can use cu121) |
| G3 | Python 3.10–3.12 + uv | `uv --version; python3 --version` | uv installed, else `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| G4 | ffmpeg on PATH | `ffmpeg -version` | present (WhisperX decodes mp3 via it on Windows; torchcodec warning is harmless) |
| G5 | Hermes MCP support + gateway running | `hermes gateway status` / `hermes mcp list` | gateway up |
| G6 | Inference endpoint | `curl -s <BASE>/models` | returns model list; note the real `id` field (often NOT the `root`/repo name) |
| G7 | Internet to hf.co, api.notion.com, *.plaud.ai | one HEAD request each | 200/301/401-ok (reachability, not auth) |

If G2 fails (no GPU): the pipeline still works with `device="cpu"` in `stage_transcribe.py` but expect ~0.3–1× realtime transcription; flag this to the human.

## Phase 2 — What you must ask the human for

Ask ONCE, in this order. Everything here is a credential or an account action only they can do:

1. **Plaud account** — they must complete a browser OAuth. You can start it (see G8 below) but cannot do the login.
2. **Hugging Face**: an account, agreement to **three** gated repos, and a **read token**:
   - https://huggingface.co/pyannote/speaker-diarization-3.1
   - https://huggingface.co/pyannote/segmentation-3.0
   - https://huggingface.co/pyannote/speaker-diarization-community-1 ← easy to miss; the installed whisperx defaults to THIS model and pulls auxiliary assets from it regardless of which `model_name` you pass
   - Token: https://hf.co/settings/tokens (Fine-grained, read is enough)
3. **Notion**: an internal integration (https://www.notion.so/my-integrations), the target database's ID, AND the exact property names for title + date of the recordings calendar/day page DB (ask for a page title pattern, e.g. `"MM/DD/YYYY- Dailies"`). They must also click *Connections → add your integration* on that database.
4. **Storage**: where recordings live (`$ROOT/recordings` default is fine), and (optional) a network share or web URL for the HTML reports.

**Secret-handling rule (learned the hard way):** tokens pasted through an agent's chat/tool layer can be silently redacted (`***`, `…`) and written to disk corrupted. Have the human edit `$ROOT/.env` themselves in a text editor, or paste into a file via their own terminal. NEVER echo secrets in tool calls; validate by length + prefix + HTTP status code only:

```python
tok = env["HF_TOKEN"]                      # from .env
assert tok.startswith("hf_") and len(tok) >= 30 and tok.encode("latin-1", errors="ignore")
# then HEAD https://huggingface.co/pyannote/segmentation-3.0/resolve/main/config.yaml
#   200=good  401=token corrupt/missing  403=token valid, gate NOT accepted
```

## Phase 3 — Environment build (exact, copy this)

```bash
mkdir -p $ROOT/{scripts,recordings,logs,prompts,docs}
uv venv $ROOT/venv-whisperx --python 3.11
```

Windows / RTX 50-series (sm_120):

```bash
uv pip install --python $ROOT/venv-whisperx/Scripts/python.exe \
   "torch==2.11.0+cu128" "torchaudio==2.11.0+cu128" \
   --index-url https://download.pytorch.org/whl/cu128
uv pip install --python $ROOT/venv-whisperx/Scripts/python.exe whisperx
# whisperx drags in CPU torchvision → transformers dies with
# "operator torchvision::nms does not exist". Re-pin AFTER whisperx:
uv pip install --python $ROOT/venv-whisperx/Scripts/python.exe \
   "torchvision==0.26.0+cu128" --index-url https://download.pytorch.org/whl/cu128
```

(Other GPUs: swap to the matching cuXXX wheels with `torch>=2.7`, same re-pin rule.)

**Verify — do not skip; a silent CPU fallback is the #1 install failure:**

```bash
python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available()); \
import whisperx; from transformers import Pipeline; t=torch.zeros(4,device='cuda'); \
from whisperx.diarize import DiarizationPipeline; print('OK')"
```

`torch.version.cuda` must NOT be `None`. If `is_available()` is False: something replaced torch with a CPU build — re-run the cu128 pins in order.

`.env` template (human fills the blanks):

```ini
HF_TOKEN=
PLAUD_...P/v1
SUMMARY_MODEL=                 # leave blank = auto-resolve from /models
NOTION_API_KEY=
NOTION_DB_ID=
NOTION_TITLE_PROP=Projects     # verify in Phase 5
NOTION_DATE_PROP=Dates
RECORDINGS_DIR=<abs path>
STATE_DB=<abs>/state.db
WHISPER_MODEL=large-v3
DIARIZE=true
```

**Windows CRLF pitfall:** write `.env` with LF endings or strip `\r` in your parser — a `WHISPER_MODEL` value of `"large-v3\r"` raises `Invalid model size`.

## Phase 4 — MCP + auth gate checks

Register the server (Hermes `config.yaml`):

```yaml
mcp_servers:
  plaud:
    command: npx
    args: ["-y", "@plaud-ai/mcp"]
    timeout: 120
    connect_timeout: 60
```

G8. `hermes mcp test plaud` → connects, 7 tools discovered.
G9. **Live auth check** (connection ≠ authentication): call `get_current_user` over stdio:

```bash
# spawn `npx -y @plaud-ai/mcp`, JSON-RPC: initialize → notifications/initialized
# → tools/call get_current_user. Expect a JSON user blob.
# If "Not authenticated": call tools/call "login" — it opens a browser.
# NOTE: `hermes mcp login plaud` does NOT work for this server (stdio-only,
# no URL) — drive the login TOOL over stdio instead. If the login call
# "times out after 2 minutes" but the user approved in the browser, the
# token persisted server-side: a second login call completes instantly.
```

G10. `list_files` returns the user's recordings (confirms the account has data).
G11. `get_file` on one id returns `presigned_url` (an S3 link) — download it, expect bytes > 1KB.

## Phase 5 — Notion gate checks

G12. `GET /v1/users/me` with the key → your bot name.
G13. `POST /v1/search` (empty query) → the target DB is visible; if not, the human forgot to connect the database to the integration.
G14. `GET /v1/databases/<id>` → record `title` and the **exact** property names + types (title-prop, date-prop). Fill `NOTION_TITLE_PROP` / `NOTION_DATE_PROP` and adapt `stage_publish.py`'s label/date logic to their convention. Ask the human how existing day pages are titled (query the DB, don't guess the pattern).
G15. All calls use base `https://api.notion.com/v1/...` — **every path needs the `/v1` prefix, including PATCH on blocks** (without it: 400 "Invalid request URL"). `Notion-Version: 2022-06-28`.

## Phase 6 — Code the pipeline

Either **copy these files from the reference machine** (`$ROOT/scripts/`) or delegate the build to a coding agent. File manifest and contracts:

| File | Contract |
|---|---|
| `plaud_client.py` | stdio JSON-RPC MCP client (spawns `npx -y @plaud-ai/mcp`). Exposes `list_files`, `get_file(file_id)`. **Parse gotcha:** large `get_file` payloads return JSON *plus a trailing plain-text note* — use `json.JSONDecoder().raw_decode()`, and treat non-dict returns as errors. Plaud API 500s are transient — fail the stage, let the retry tick fix it. |
| `pipeline_state.py` | SQLite: `recordings` + `stage_state(recording_id,stage,status,detail,updated_at)`; `stage='download'` `detail`= audio path — every later stage reads paths from here. |
| `stage_download.py` | get_file → presigned_url → save `<RECORDINGS_DIR>/<YYYY>/<id>/<name>.mp3` (flat by year; YYYY from local start_at; legacy fallback to `<RECORDINGS_DIR>/<id>/`). Skip if audio exists on disk. `get_file` arg is `file_id` (not `id`). Never touch get_transcript/get_note. RECORDINGS_DIR may be a network share (must be writable from BOTH terminal and cron context — check the logon session; drive maps are per-session). |
| `stage_transcribe.py` | whisperx **3.8.x API** (older than 3.9 docs): `model.transcribe(audio, batch_size=8)` (VAD + language detect run inside); `whisperx.align(segments, align_model, meta, audio, "cuda", return_char_alignments=False)` — NOT `return_char_indices`. Diarization: `from whisperx.diarize import DiarizationPipeline; pipe = DiarizationPipeline(token=env["HF_TOKEN"], device="cuda")` — leave `model_name` at its community-1 default — then `whisperx.assign_word_speakers(diar, result)`. GPU models are memoized module-level (`_get_whisper/_get_align/_get_diar`) so a batch in one pipeline.py process loads large-v3 (~33 s) once, not per recording. Sanitize the token first (see G2 rules) and **never fail the stage on diarization errors — warn and continue with `speaker: null`.** Emit `{audio, duration_s, language, diarized, segments:[{start,end,speaker,text,words[]}]}`. |
| `stage_summarize.py` | resolve model id dynamically from `<VLLM>/models` (prefer `SUMMARY_MODEL` if present); POST chat/completions with `response_format:{type:"json_schema",json_schema:{name,schema,strict:true}}`, `temperature:0.2`, `chat_template_kwargs:{enable_thinking:false}`. User message = compact `[MM:SS] SPEAKER: line` transcript + system = `prompts/summary.md`. Validate against schema; one retry appending the validation error. ⚠ some vLLM stacks stream into `reasoning` and leave `content` empty — guard. |
| `stage_render.py` | regex-replace the `<script id="report-data">` JSON in the template; escape `</script`→`<\/script`; merge same-speaker segments with gap <2s into turns (normalize `None`→`SPEAKER_?` BEFORE comparing, or nothing merges); output `<audio-stem>.html`. ⚠ treat `start_at` as naive **UTC** and convert to local for display. |
| `stage_publish.py` | per-day pages: query DB by date-prop `equals`; create page (title-prop + date) if missing; find-or-create a `🎙 Recordings` heading; append one **toggle per recording at PAGE level** (heading_2 blocks reject children: "Block does not support children"); inside toggle: **inline report preview — embed block**: `POST /v1/file_uploads` (2025-09-03) → **multipart/form-data POST to the returned /send URL — the send request MUST also carry `Notion-Version: 2025-09-03`** (routing it around the versioned helper 400'd every backfill row once) → poll until status `uploaded` → append with `position:{type:"start"}` (start-insert only exists in 2025-09-03+; 2022 rejects `before_id` entirely) — uploads expire ~1h if never attached, so attach in the same call; then TL;DR / takeaways / decisions / to_do items / report path as plain-text fallback (**Notion link objects reject `file://` URLs**) / nested transcript toggle (paragraph per segment). Chunk appends ≤90 children, sleep 0.35s (≈3 rps); honor 429 Retry-After. Store `toggle_id` + `embed_id` in state detail; re-publish = archive toggle **only** (archiving a parent archives its subtree — never PATCH children after, and never archive the child ids: 400 "archived ancestor"). Orphan-sweep: archive childless toggles whose label prefix matches the recording. Use `%I:%M %p` for 12-hour labels if the human prefers. |
| `pipeline.py` | discovery = unseen ids (pages until a known id) ∪ **anything without a completed `publish` row** (state-proof; don't rely on `failed` rows — cleared state would make recordings invisible forever). Reads `.env` fresh per process start — a long-lived sweep that predates a `.env` fix must be restarted. `--dry-run` stops before publish; `--id` bypasses the lock; `.tick.lock` exit if held <4h. |

Template, schema, and prompt files (`meeting-report-template.html`, `summary.schema.json`, `prompts/summary.md`) are content-agnostic — copy them as-is from the reference machine or have the coding agent regenerate them from the contracts above.

### Claude Code delegation prompt

> Build the pipeline described in docs/SETUP-GUIDE (Phase 6 file manifest) into `$ROOT/scripts/`.
> Constraints that must not be violated: never call Plaud `get_transcript`/`get_note`; whisperx 3.8.x API signatures as listed; all Notion URLs include `/v1`; stage functions take `(conn, rid, ...inputs..., env, log)` and are idempotent via `pipeline_state`.
> Acceptance: every gate check in Phase 7 prints PASS on this machine; `pipeline.py --id <a real recording id>` downloads → transcribes (speaker labels present) → summarizes (validates vs schema) → renders (report-data parses as JSON) → publishes a toggle into the correct day page.

Delegate per-stage (one stage + its gate per task) rather than all-at-once; it localizes failures and lets you run each gate immediately.

## Phase 7 — Stage gate checks (the acceptance test)

```bash
# G16 download
<venv-py> -c "import sys;sys.path.insert(0,'scripts');import pipeline_state,stage_download;\
from plaud_client import PlaudMCP;c=pipeline_state.db('state.db');m=PlaudMCP();\
info=m.get_file('<REAL_ID>');print(stage_download.run(c,info,'recordings'))"   # -> audio path, >1KB
# G17 transcribe (diarization must produce speakers)
<venv-py> scripts/test_stage2.py    # expect "N segments, >=2 speakers"
# G18 summarize
<venv-py> -c "... stage_summarize.run(...)"   # expect: keys == schema.required; no ValidationError
# G19 render
python - <<'EOF'  # parse the report-data block back out: json.loads must succeed
EOF
# G20 publish — inspect via GET /v1/blocks/<page>/children:
#   toggle under the page; nested: paragraphs, to_do blocks, transcript toggle
```

## Phase 8 — Cron + on-demand

Register with Hermes' `cronjob` tool: full sweep `every 15m`, publish-drain `every 5m`, both `no_agent: true` with `script` tick wrappers, `deliver="all"` (or the human's channel).

⚠ **On Windows the cron scheduler has NO bash on PATH** (only the interactive terminal does). `.sh` scripts fail with "bash not found on PATH" while everything looks configured correctly. Write tick wrappers in **Python** (stdlib only). **Also the scheduler inherits `PYTHONPATH` from the agent's own venv**, which shadows the pipeline venv's packages (classic symptom: `ImportError: tokenizers>=x,<=y is required… found z` inside transcribe, while running the same command from a terminal works). Tick wrappers must `environ.pop("PYTHONPATH", None); environ.pop("PYTHONHOME", None)` before launching the pipeline. A tick prints a one-line summary on work/failures and NOTHING when idle (empty stdout = no delivery). The overlap lock makes ticks no-op while a long run holds it.

Manual on-demand trigger: `cronjob(action="run", job_id=...)`, or the human just messages the agent "run the plaud pipeline" and the agent executes the same.

## Phase 9 — Pitfall index (all real, all cost hours; verify each in order)

1. `hermes mcp test plaud` passes while tools return "Not authenticated" — check auth with a live `get_current_user`.
2. Chat/tool-layer secret redaction corrupts tokens written via tool calls — human edits `.env` directly; verify by HTTP code (401 corrupt / 403 gate).
3. Three pyannote gates, not two — community-1 is the third and whisperx's default.
4. whisperx drags CPU torchvision over the CUDA one → `torchvision::nms does not exist`. Re-pin cu128 torchvision AFTER whisperx; verify `torch.version.cuda`.
5. Transitive deps can silently downgrade cu128 torch to CPU mid-install. Re-check after every install step.
6. whisperx 3.8.x ≠ 3.9+ API (`return_char_alignments`, `DiarizationPipeline`, no `whisperx.diarize()`).
7. `get_file` = `file_id` param; JSON+trailing-note payload; 500s transient.
8. Naive `start_at` is UTC — convert to local for labels, day-page dates, and HTML meta.
9. Long sweeps cache `.env` at process start — restart them after token fixes; don't trust "stale" transcripts with 0 speakers (re-diarize pass pattern).
10. CRLF in `.env` values (`"large-v3\r"`).
11. Notion: `/v1` on every path; `file://` rejected in links; heading_2 rejects children; toggles accept nested PATCH; parent-archive covers subtree; 100-children/2000-char limits; 429 pacing.
12. Discovery keyed on `failed` state rows is not crash-proof — key it on "no completed publish row".
13. `hermes config` protects `config.yaml` from agent edits — use `hermes config set` or the human edits.
14. If vLLM streams into `reasoning` with empty `content`, guard for it (some builds).

## Phase 10 — Final acceptance checklist

- [ ] Fresh recording on the Plaud device appears in Notion within ~20–30 min, under the right day page, 12h-clock toggle, ≥2 speaker labels, valid report HTML
- [ ] Kill the sweep mid-run and restart → completes the same recording once, no duplicate toggles
- [ ] Remove `HF_TOKEN` → transcribe warns and still publishes (degrades, never fails)
- [ ] Corrupt the token to `hf_…` style truncation → same warn-and-degrade (sanitizer path)
- [ ] Expire Plaud auth (logout tool) → next tick alert names "not authenticated"; re-login flow documented works
- [ ] Both cron jobs deliver nothing on idle ticks

**Reference implementation:** this machine keeps the working scripts at `$ROOT/scripts/` (plus `rediarize.py`, `publish_backlog.py`, `republish_labels.py` one-off tools) and the operational lore in the Hermes skill `plaud-notion-pipeline`.
