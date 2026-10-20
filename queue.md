# Dubbio — 03 Queue and job payloads

Stack: Redis 7 + BullMQ. One worker process in MVP (single VPS). Names below match `02-database.md` (tables, enums) and `03-api.md` (endpoints). Postgres is the record of truth for every job; Redis only carries the work.

## 0. Queues and job types

| Queue | Job name | Producer | Concurrency | Attempts (BullMQ) | Backoff | jobId (idempotency key) |
|---|---|---|---|---|---|---|
| `voices` | `voice.clone` | `POST /voices`, `POST /voices/:id/retry` | 1 | 3 | exp 30 s | `voice:{voice_id}:clone` |
| `voices` | `voice.delete` | `DELETE /voices/:id` | 1 | 5 | exp 30 s | `voice:{voice_id}:delete` |
| `pipeline` | `pipeline.run` | `upload/complete`, `voices/confirm`, `retry` | 1 | 2 (crash resume only) | fixed 10 s | `pipeline:{project_id}` |
| `render` | `segment.regenerate` | review `edit`, `characters/:id/voice`, `versions/:id/retry` | 2 | 3 | exp 15 s | `version:{version_id}` |
| `render` | `project.export` | end of `segment.regenerate` batch, `revert`, `POST /exports`, pipeline `export` step | 2 (1 per project by rule) | 3 | exp 15 s | `export:{export_id}` |
| `maintenance` | `project.purge` | `DELETE /projects/:id` | 1 | 5 | exp 60 s | `purge:{project_id}` |
| `maintenance` | `cleanup.uploads` | repeatable, hourly | 1 | 1 | — | `cleanup.uploads` |
| `maintenance` | `cleanup.otp_sessions` | repeatable, daily 03:00 | 1 | 1 | — | `cleanup.otp_sessions` |
| `maintenance` | `cleanup.retention` | repeatable, daily 03:30 | 1 | 1 | — | `cleanup.retention` |
| `maintenance` | `usage.spend_check` | repeatable, hourly | 1 | 1 | — | `usage.spend_check` |
| `maintenance` | `jobs.stuck_sweep` | repeatable, every 15 min | 1 | 1 | — | `jobs.stuck_sweep` |

Common BullMQ options: `removeOnComplete: true`, `removeOnFail: true` (history lives in `step_runs`, `segment_versions`, `exports`, `voices`, Sentry — not in Redis, so jobIds can be reused after a failure and a Redis flush loses nothing durable), `lockDuration: 60000`, `stalledInterval: 30000`, `maxStalledCount: 2`. The `pipeline` queue has concurrency 1 because `separate` (Demucs on CPU) must not run twice at once on one machine and dominates the machine anyway; raise it when separation moves to a provider.

Two kinds of retry, never confused:
- **BullMQ attempts** cover process death (stalled lock, crash). The job restarts and resumes from the DB status (every step and version is idempotent).
- **Step/call retries** inside the job cover provider errors: 3 attempts per step with 10 s → 30 s → 90 s, `step_runs.attempts` incremented; per-segment calls retry 3 times with 5 s → 15 s → 45 s.

Every job payload carries `studio_id` so logs, metrics and R2 keys never need a lookup, and `enqueued_by` for audit.

## 1. External provider limits (in-process limiters, one worker)

| Provider / tool | Limiter | Per-call timeout | On 429 / 5xx |
|---|---|---|---|
| ElevenLabs TTS | 3 concurrent, 1 queue shared by `pipeline` and `render` | 60 s | honour `Retry-After`, else 5/15/45 s; after 3 failures the segment version fails |
| ElevenLabs STT (Scribe) | 1 concurrent | 5 min | step retry |
| ElevenLabs IVC create / delete | 1 concurrent | 2 min | job retry |
| LLM (analyze, translate) | 4 concurrent | 90 s | per-batch retry; `analyze` falls back to neutral |
| Demucs (separation) | 1 concurrent | 15 min (spike value) | step retry |
| ffmpeg / ffprobe | 2 concurrent | 5 min per invocation | step retry |

Provider price table lives in config; every paid call writes one `usage_events` row (`kind`, `quantity`, `cost_usd`). When the worker scales past one process, the limiters move to a Redis token bucket (BullMQ `limiter` on a dedicated `tts` queue is the simplest upgrade).

## 2. Job specifications

### 2.1 `voice.clone`

Payload
```json
{ "voice_id": "uuid", "studio_id": "uuid", "enqueued_by": "user:uuid" }
```
Steps: `voices.status := cloning` → download `sample_key` → ffprobe (reject if < 30 s or > 10 min: `failed`, no provider call) → ffmpeg normalise to mono 44.1 kHz MP3 → ElevenLabs IVC create (name = `voices.name`, label `studio_id`) → set `provider_voice_id`, `sample_duration_ms`, `status := ready` → event `voice.updated`.
Result `{ "voice_id": "uuid", "provider_voice_id": "…", "sample_duration_ms": 92000 }`.
Idempotency: if `provider_voice_id` is already set, skip the provider call. Timeout 5 min total. Final failure: `status := failed`, `error` set, event.

### 2.2 `voice.delete`

Payload `{ "voice_id": "uuid", "studio_id": "uuid", "enqueued_by": "user:uuid" }`.
Steps: ElevenLabs delete (404 counts as success) → `deleted_at := now()`, `status` stays `deleting` → event. Final failure: `error` set, row stays `deleting`; the sweep retries daily.

### 2.3 `pipeline.run`

Payload
```json
{ "project_id": "uuid", "studio_id": "uuid",
  "from_step": "validate | translate | <failed_step>",
  "enqueued_by": "upload_complete | voices_confirm | retry" }
```
The job runs steps in order from `from_step` to the end of the current phase, then returns. Phase 1 = `validate, separate, transcribe, analyze` → `needs_voices`. Phase 2 = `translate, synthesize, mix, export` → `completed`. Before each step it reloads the project; `deleted_at` set → stop silently. A step already `done` is skipped (crash resume).

Per step: `step_runs.status := running`, `started_at`, `attempts + 1`, event `step.updated`; on success `done`, `finished_at`, `output`; on final failure `failed`, `error`, and `projects.status := failed`, `failed_step`, `error`, event `project.updated`.

| Step | Reads | Writes | Output JSON (`step_runs.output`) | Timeout | Failure policy |
|---|---|---|---|---|---|
| `validate` | `assets(source_video)` | `projects.duration_ms`, `source_meta`; `usage_events(video_minutes)` | `{ "duration_ms", "width", "height", "fps" }` | 2 min | no retry; fail with `error` in `{too_long, not_mp4, no_audio, quota_exceeded}` |
| `separate` | `source_video` | `assets(voice_track)`, `assets(background_track)`; `usage_events(separation_seconds)` | `{ "voice_key", "background_key", "seconds" }` | 15 min | 3 retries |
| `transcribe` | `voice_track` | `assets(transcript)` raw JSON; `characters` (one per diarization speaker, `sort_order` by first appearance); `segments` (`idx`, `start_ms`, `end_ms`, `source_text`, `character_id`); `usage_events(stt_seconds)` | `{ "segment_count", "character_count" }` | 10 min | 3 retries; re-run deletes and recreates characters/segments (idempotent because nothing references them yet) |
| `analyze` | `segments`, `transcript` | `projects.context`, `segments.emotion`, `segments.addressee_character_id`, `segments.analysis`, `characters.analysis`; `usage_events(llm_tokens)` | `{ "fallback": false, "tokens" }` | 5 min | 3 retries, then `done` with `fallback: true`, all emotions `neutral`, addressee null |
| `translate` | `segments`, `characters` (gender from voice), `projects.context`, `target_lang`, `target_dialect`, slot duration | `segment_versions` v1 (`reason: initial`, `status: pending`, `voice_id` resolved from `characters.voice_id`, `emotion` from segment); `usage_events(llm_tokens)` | `{ "translated", "failed_segment_ids": [] }` | 10 min | batches of 20 segments in dialogue order with 2 batches of context overlap; a failed batch retries alone; after retries, any segment still without a version fails the step |
| `synthesize` | v1 versions with `status != ready` | `segment_versions` audio columns, `fit_status`, `status`; `segments.current_version_id`; `usage_events(tts_characters)` | `{ "generated", "too_long", "failed_segment_ids": [] }` | 15 min | per-segment retries; if > 20 % of segments remain `failed`, the step fails; otherwise `done`, failed segments are listed and mixed with the original voice (§2.4 fallback) |
| `mix` | ready versions, `background_track`, `voice_track` (fallback) | `assets(dubbed_track)` | `{ "dubbed_key", "fallback_segment_ids": [] }` | 5 min | 3 retries |
| `export` | `dubbed_track`, `source_video` | `exports` v1 (`trigger: pipeline`, `content_version`, `manifest`) then the same code path as `project.export`; `projects.current_export_id`, `completed_at`, `status := completed` | `{ "export_id", "video_key" }` | 5 min | 3 retries |

Synthesize detail, per segment version: TTS (model from `projects.render_settings.tts_model`, emotion → v3 audio tag, or ignored on v2) → ffprobe `raw_duration_ms` → required = `end_ms - start_ms`; if `raw ≤ required` pad with silence; else `speed_factor = raw / required`, capped at `render_settings.fit_max_speed` (1.25) → ffmpeg `atempo` → `fitted_duration_ms`; `fit_status = ok` if fitted ≤ required else `too_long` → upload `segments/{segment_id}/v{version_no}.mp3` → `status := ready`, `segments.current_version_id := id`, event `segment_version.updated`. Idempotent: the R2 key is deterministic and the row is re-written.

Result `{ "project_id": "uuid", "ended_at_step": "analyze | export", "status": "needs_voices | completed | failed" }`.

### 2.4 `segment.regenerate`

Payload
```json
{ "project_id": "uuid", "studio_id": "uuid", "segment_id": "uuid", "version_id": "uuid",
  "batch_id": "uuid | null", "enqueued_by": "user:uuid" }
```
Steps: load version (`pending` or `failed`; else exit) → `status := generating` → the same per-segment synthesize routine as §2.3 → on success `ready`, `segments.current_version_id := version_id`, `review_status := pending`, `projects.content_version + 1` (one transaction) → `usage_events(tts_characters)` → event → **export rule** (§2.5).
Result `{ "version_id", "status", "fit_status", "audio_key", "tts_characters" }`.
Final failure: `status := failed`, `error`, event; `current_version_id` untouched; the user retries with `versions/:id/retry` (re-adds the same jobId) or edits again.
Batch rule: when `batch_id` is set, the export rule runs only if no other version of that batch is `pending/generating`, so a character voice change renders once.

### 2.5 `project.export` and the export rule

Export rule (called by `segment.regenerate`, `revert`, `POST /exports`, and by a finishing export): in one transaction, `INSERT exports (project_id, version_no = max+1, trigger, content_version = projects.content_version, manifest = current versions, status = pending) WHERE NOT EXISTS (exports with status IN (pending, rendering) for the project)`; if a row was inserted, add job `export:{export_id}`. At most one export per project is in flight; nothing is lost because the finishing export re-checks `projects.content_version > exports.content_version` and applies the rule again.

Payload `{ "project_id": "uuid", "studio_id": "uuid", "export_id": "uuid", "enqueued_by": "…" }`.
Steps: `status := rendering` → build the dubbed track from `manifest` (for each segment: ready version audio at `start_ms`; segment with `null` in the manifest → slice of the original `voice_track` for `[start_ms, end_ms]`) over `background_track` → mux with `source_video` (video stream copied, audio AAC 192 kbps) → upload `exports/v{version_no}.mp4` → `status := ready`, `video_key`, `size_bytes`, `projects.current_export_id := export_id` → `usage_events(export_render)` → event `export.updated` → apply the export rule.
Result `{ "export_id", "video_key", "size_bytes" }`. Final failure: `status := failed`, `error`, event; `current_export_id` unchanged, so download keeps working.

### 2.6 `project.purge`

Payload `{ "project_id": "uuid", "studio_id": "uuid", "enqueued_by": "user:uuid" }`.
Steps: abort any open multipart upload → list R2 prefix `studios/{studio_id}/projects/{project_id}/` → delete in batches of 1000 → hard-delete `projects` (cascades) → done. Only runs when `deleted_at` is set. Idempotent by construction (prefix listing).

### 2.7 Repeatable maintenance jobs

| Job | Work |
|---|---|
| `cleanup.uploads` | `uploads` with `status = pending` and `expires_at < now()`: abort multipart on R2, `status := expired`; if the project is still `uploading`, soft-delete it and enqueue `project.purge`. |
| `cleanup.otp_sessions` | delete `otp_codes` older than 24 h and `sessions` expired or revoked for > 7 days. |
| `cleanup.retention` | per `02-database.md` §7: intermediate assets 30 days after `completed_at`; non-current version audio after 30 days (row kept, `audio_key := null`); non-current exports after 7 days. |
| `usage.spend_check` | sum `usage_events.cost_usd` for the calendar month; at ≥ 80 % of the configured cap send an ops alert once per day; at ≥ 100 % set Redis key `spend_cap_reached` (TTL to month end). The API checks that key on `upload/complete`, `voices/confirm`, `POST /voices` and every review write and answers `403 quota_exceeded` with `details.quota = "global_spend"`. |
| `jobs.stuck_sweep` | `step_runs.status = running` with `started_at` older than 2× the step timeout and no active BullMQ job for the project → mark step and project `failed` (`error: stalled`); same for `segment_versions.generating` older than 10 min and `exports.rendering` older than 15 min. This is the dead-letter mechanism: failures land in the tables the UI already shows, with a `retry` endpoint for each. |

Worker-level `failed` event handler (job threw outside step logic or exhausted BullMQ attempts): same marking as the sweep, immediately.

## 3. Sequences

### 3.1 Happy path

```mermaid
sequenceDiagram
  participant W as Web
  participant A as API
  participant Q as Redis/BullMQ
  participant K as Worker
  participant P as Postgres
  participant E as ElevenLabs / LLM
  W->>A: POST /projects/:id/upload/complete
  A->>P: project queued, 8 step_runs pending
  A->>Q: pipeline.run {from_step: validate}
  A-->>W: 200 (SSE: project.updated queued)
  Q->>K: pipeline.run
  K->>P: analyzing; validate → separate (Demucs) → transcribe
  K->>E: STT with diarization
  K->>P: characters + segments; analyze (LLM) → context, emotion
  K->>P: needs_voices (SSE)
  W->>A: PUT /characters/:id/voice ×N, POST /voices/confirm
  A->>Q: pipeline.run {from_step: translate}
  Q->>K: pipeline.run
  K->>E: translate (LLM batches)
  K->>P: segment_versions v1 pending
  K->>E: TTS per segment (3 concurrent)
  K->>P: versions ready, current_version_id, mix, exports v1 ready
  K->>P: completed, current_export_id (SSE)
  W->>A: GET /projects/:id/download
```

### 3.2 Partial failure and resume

- **Provider down during `transcribe`**: 3 step retries fail → `step_runs(transcribe).failed`, project `failed`, `failed_step = transcribe`. `POST /retry` → `pipeline.run {from_step: transcribe}`; `validate` and `separate` are skipped because they are `done`.
- **Two segments fail in `synthesize`**: per-segment retries exhausted → those versions `failed`; 2/42 < 20 % → step `done` with `failed_segment_ids`; `mix` uses the original voice for those spans; project `completed`; the review page shows the two segments as `failed` with `versions/:id/retry`.
- **Worker crashes mid-`synthesize`**: BullMQ marks the job stalled and re-runs it (attempt 2); the job skips done steps and the synthesize step only processes versions whose `status != ready`; audio keys are deterministic so nothing is duplicated.
- **Quota exhausted**: `validate` writes `usage_events(video_minutes)` only if the monthly sum stays within quota; otherwise the project fails with `error = quota_exceeded` before any paid call.

### 3.3 Reviewer rejects a segment

```mermaid
sequenceDiagram
  participant W as Web
  participant A as API
  participant Q as Redis/BullMQ
  participant K as Worker
  participant P as Postgres
  W->>A: PATCH /review/segments/:sid {review_status: rejected}
  W->>A: POST /review/segments/:sid/edit {translated_text}
  A->>P: regeneration_count+1 (atomic), version v2 pending
  A->>Q: segment.regenerate {version_id}
  A-->>W: 202 (segment.pending_version = v2)
  Q->>K: segment.regenerate
  K->>P: v2 generating → ready; current_version_id=v2; review_status=pending; content_version+1
  K->>P: export rule → exports v2 pending
  K->>Q: project.export {export_id}
  Q->>K: project.export
  K->>P: exports v2 ready; current_export_id (SSE export.updated)
  W->>A: PATCH /review/segments/:sid {review_status: approved}
  Note over W,A: Undo = POST /revert {version_id: v1} → content_version+1 → export rule only (no TTS, not counted)
```

While `exports v2` renders, `GET /download` still returns `exports v1` with `is_stale: true`.
