# Dubbio — 04 Backend file and folder architecture

One NestJS codebase, two processes (`api`, `worker`), one Docker image with two commands. pnpm, TypeScript strict, Prisma. Module and folder names match `03-api.md` modules and `03-queue.md` job names.

## 1. Tree

```
dubbio-backend/
├── package.json  pnpm-lock.yaml  tsconfig.json  tsconfig.build.json  nest-cli.json
├── .env.example                      # every variable the config schema validates
├── Dockerfile                        # one image; CMD overridden per process
├── docker-compose.yml                # api, worker, postgres, redis, bull-board (dev + single-VPS prod)
├── .github/workflows/ci.yml          # lint, typecheck, unit, e2e (postgres+redis services), build, push, deploy staging
├── docs/                             # 01-overview … 06-risks (these documents)
├── prisma/
│   ├── schema.prisma                 # enums and models from 02-database.md
│   ├── migrations/                   # generated; sequential; one open migration PR at a time
│   └── seed.ts                       # dev studio, user, two mock voices
├── scripts/
│   ├── provider-spike/               # Sprint 0: demucs-cpu, v3-timing, arabic-quality, scribe-diarization
│   └── backup-postgres.sh            # nightly pg_dump → R2, 7 days
├── src/
│   ├── main.api.ts                   # bootstrap HTTP app (AppApiModule)
│   ├── main.worker.ts                # bootstrap worker app (AppWorkerModule), no HTTP except /health
│   ├── app.api.module.ts             # imports: config, infra, every modules/* http module
│   ├── app.worker.module.ts          # imports: config, infra, providers, pipeline, jobs, modules/*/domain
│   │
│   ├── config/                       # typed env config (zod schema), price table, limits (sizes, timeouts, quotas)
│   │
│   ├── common/                       # no internal deps. Shared primitives only.
│   │   ├── errors/                   # AppError + error code enum (03-api §1.2), HTTP exception filter
│   │   ├── auth/                     # JwtAuthGuard, CurrentUser decorator, AuthContext {user_id, studio_id}
│   │   ├── idempotency/              # Idempotency-Key interceptor (Redis), body hash
│   │   ├── rate-limit/               # sliding-window guard (Redis), limit table
│   │   ├── pagination/               # cursor encode/decode, list response envelope
│   │   ├── validation/               # DTO validation pipe config, shared validators (uuid, enum, bcp47)
│   │   └── logging/                  # pino setup, request/job id correlation
│   │
│   ├── infra/                        # adapters to our own infrastructure
│   │   ├── prisma/                   # PrismaService, transaction helper, TenantScoped base repository (studio_id first arg)
│   │   ├── redis/                    # ioredis clients (commands, pub, sub)
│   │   ├── queue/                    # BullMQ module: queue names, job names, Queue producers, payload + result types (the contract from 03-queue.md)
│   │   ├── storage/                  # R2 client: presign PUT/GET, multipart create/parts/complete/abort/listParts, head, deletePrefix, key builder (02-database §7)
│   │   ├── events/                   # DomainEvent types (03-api §11), Redis pub/sub publisher + subscriber
│   │   └── email/                    # transactional email port + provider adapter + console adapter
│   │
│   ├── providers/                    # external AI/media services behind interfaces, each with a mock
│   │   ├── tts/                      # TtsProvider interface; elevenlabs-v3, elevenlabs-v2, mock; emotion → audio tag map
│   │   ├── stt/                      # SttProvider; elevenlabs-scribe (diarization), mock
│   │   ├── llm/                      # LlmProvider (JSON-mode chat); openai-compatible adapter, mock; prompts/ (analyze, translate)
│   │   ├── voice-clone/              # VoiceCloneProvider; elevenlabs-ivc, mock
│   │   ├── separation/               # SeparationProvider; demucs (local CLI), mock; provider chosen by config
│   │   ├── media/                    # ffmpeg/ffprobe wrappers: probe, normalise, atempo, pad, mix, mux; temp-dir handling
│   │   ├── limiter/                  # per-provider concurrency + retry/backoff (03-queue §1)
│   │   └── providers.module.ts
│   │
│   ├── modules/                      # domain modules. Each: domain/ (services, repositories, no HTTP) and http/ (controllers, DTOs)
│   │   ├── auth/                     # otp request/verify, sessions, refresh rotation, logout
│   │   ├── studios/                  # studio read/update, quotas, usage summary (reads usage domain)
│   │   ├── usage/                    # usage_events writer, monthly sums, spend-cap check
│   │   ├── voices/                   # voices CRUD, sample upload, slot check; enqueues voice.clone / voice.delete
│   │   ├── projects/                 # projects CRUD, uploads (multipart), characters, voices/confirm, retry, steps; enqueues pipeline.run
│   │   ├── segments/                 # segment + version read models, derived state, presigned audio URLs
│   │   ├── review/                   # review summary, review_status, edit, revert, version retry, character voice change; regeneration limit; enqueues segment.regenerate
│   │   ├── exports/                  # export rule (03-queue §2.5), exports list, download URL; enqueues project.export
│   │   ├── events/                   # SSE ticket + /events stream controller (api only), subscribes infra/events
│   │   └── health/                   # /health, /health/ready
│   │
│   ├── pipeline/                     # worker only: pipeline.run processor
│   │   ├── pipeline.processor.ts     # loads project, iterates steps from from_step, step-run bookkeeping, phase end
│   │   ├── step-runner.ts            # retry/backoff/timeout wrapper around one step, writes step_runs, publishes step.updated
│   │   ├── steps/
│   │   │   ├── validate.step.ts
│   │   │   ├── separate.step.ts
│   │   │   ├── transcribe.step.ts
│   │   │   ├── analyze.step.ts
│   │   │   ├── translate.step.ts
│   │   │   ├── synthesize.step.ts    # delegates per segment to synthesis/
│   │   │   ├── mix.step.ts
│   │   │   └── export.step.ts        # creates exports v1 and calls exports domain render
│   │   └── synthesis/                # per-segment TTS + fit routine shared with jobs/segment-regenerate
│   │
│   ├── jobs/                         # worker only: every other processor from 03-queue.md
│   │   ├── voice-clone.processor.ts
│   │   ├── voice-delete.processor.ts
│   │   ├── segment-regenerate.processor.ts
│   │   ├── project-export.processor.ts
│   │   ├── project-purge.processor.ts
│   │   ├── maintenance/              # cleanup.uploads, cleanup.otp_sessions, cleanup.retention, usage.spend_check, jobs.stuck_sweep
│   │   └── jobs.module.ts            # registers processors + repeatable schedules
│   │
│   └── admin/                        # Bull Board mounted in api under basic auth (dev/ops only)
│
└── test/
    ├── e2e/                          # API flows against real postgres+redis, mock providers (per module)
    ├── worker/                       # pipeline and job tests with mock providers and a local R2 stub (MinIO or in-memory)
    ├── fixtures/                     # 10-second mp4, wav samples, transcript JSON, LLM canned responses
    └── setup/                        # test db reset, factories
```

Unit tests sit next to the code (`*.spec.ts`). Mock providers are first-class: `PROVIDERS_MODE=mock` in `.env` runs the full pipeline on a laptop in seconds and is what CI uses.

## 2. Layer dependency rules

```
config, common            → nothing internal
infra                     → config, common
providers                 → config, common, infra/storage, infra/redis (limiter state later)
modules/*/domain          → config, common, infra, other modules' domain (exported services only)
modules/*/http            → own domain only
pipeline, jobs            → providers, infra, modules/*/domain
app.api.module            → modules/*/http, infra, admin
app.worker.module         → pipeline, jobs, providers, infra, modules/*/domain
```

Hard rules, enforced with `dependency-cruiser` in CI:
1. Nothing under `modules/` imports `pipeline/`, `jobs/` or `providers/`. The API never calls a provider; it only enqueues (`infra/queue` producers). The single exception is `infra/storage` for presigning and HEAD, which is our infrastructure, not an AI provider.
2. No module imports another module's `repository`; it imports the other module's exported domain service. Example: `review` calls `exports.domain.ExportRule.apply(project_id, trigger)`, never `exports` tables.
3. `pipeline/` and `jobs/` never contain HTTP, DTOs or guards; they call domain services for every DB write that has a rule (regeneration count, export rule, content_version). Steps may write their own tables through the domain repositories exposed for that purpose (`projects.domain.StepRunRepository`, `segments.domain.SegmentWriter`).
4. `providers/` never import Prisma. Inputs are file paths or buffers plus plain parameters; outputs are plain results. This is what makes mocks and spikes trivial.
5. Job payload and event types live in `infra/queue` and `infra/events` only, and are imported by both producers and processors — one contract, versioned with the code.
6. `common/` never imports from anywhere else; if it needs Prisma or Redis, it belongs in `infra/`.

Tenancy rule (test-enforced): every repository method on a studio-owned table takes `studio_id` as its first parameter; a lint test greps `TenantScoped` repositories for a missing `studio_id` in `where`.

## 3. Ownership for four developers

| Dev | Owns (folders) | Delivers | Depends on |
|---|---|---|---|
| **A — Platform** | `config`, `common`, `infra/*`, `modules/auth`, `modules/studios`, `modules/usage`, `modules/events`, `modules/health`, `admin`, `prisma/`, Docker, CI | repo skeleton, Prisma schema from 02, queue/event contracts from 03-queue, auth, SSE, rate limits, idempotency, quotas | — (goes first) |
| **B — Projects** | `modules/projects`, `modules/segments`, `modules/exports`, `infra/storage` | multipart upload, characters, confirm/retry, segments read model, export rule, download, purge producer | A's skeleton |
| **C — Providers & voices** | `providers/*`, `modules/voices`, `jobs/voice-*`, `scripts/provider-spike` | Sprint 0 spikes, provider interfaces + mocks + real adapters, voice library, limiter | A's queue contract |
| **D — Pipeline & review** | `pipeline/*`, `jobs/segment-regenerate`, `jobs/project-export`, `jobs/maintenance`, `modules/review` | the eight steps, synthesis routine, regeneration, re-export, review endpoints | B's segment/export domain, C's provider interfaces (mocks first) |

Sequencing to avoid conflicts:
- Week 1 (A, with everyone reviewing): `schema.prisma`, `infra/queue` payload types, `infra/events` types, error codes, DTO conventions, `.env.example`. These files are frozen after week 1; changes go through a PR that A owns.
- C ships mock providers before real ones so D's pipeline runs end-to-end on mocks from week 2.
- One owner per folder; cross-folder edits are PRs to the owner. Prisma migrations: one open migration PR at a time, named `NNN_<topic>`, rebased before merge.
- The pipeline `export.step.ts` and `jobs/project-export` share `exports.domain.RenderService` (B); D consumes it, B owns it.

## 4. Process and runtime layout

| Process | Command | Imports | Ports |
|---|---|---|---|
| `api` | `node dist/main.api.js` | `app.api.module` | 3000 (behind Cloudflare), `/admin/queues` bound to localhost |
| `worker` | `node dist/main.worker.js` | `app.worker.module` | 3001 `/health` only |

Both read the same `.env`; `PROCESS_ROLE` is not needed because the entrypoint decides. The worker container image adds `ffmpeg` and the Demucs runtime (Python + model weights) in a separate build stage so the API image stays small. Horizontal path later: run more `worker` containers and move provider limiters to Redis (03-queue §1); nothing in the layout changes.
