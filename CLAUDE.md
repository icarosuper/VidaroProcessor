# CLAUDE.md — VidroProcessor

Async Go worker for VidroApi. Pull video IDs from Redis queue, download from MinIO, run 7-step FFmpeg pipeline, upload artifacts back. Stateless — scale via more instances same Redis.

## Always-on rules

- **Language**: all code, logs, errors, comments, commits, docs in English. No Portuguese in source.
- **Error wrapping**: `fmt.Errorf("context: %w", err)` — never `%v`.
- **Required config**: required env vars use `notEmpty` (caarlos0/env); optional use `envDefault`. Mirror every new var in `.env-example`.
- **Shared contract with VidroApi**: queue names, MinIO paths, webhook payload. Changes need coordinated tag + deploy both repos.
- **Branching**: commit straight to `master` by default; `feature/<topic>` only for large multi-commit features, after asking the user (see "Onde commitar"). `master` always deployable; prod deploys from `vX.Y.Z` tags.
- **Only create files when necessary.** No `*.md`/README unless asked.

## Running locally

```bash
cp .env-example .env
docker-compose up -d redis minio          # this repo's compose (processor only)
# or the whole Vidro stack, from the parent dir: cd .. && docker compose up -d --build
go run main.go
```

## Running tests

```bash
go test ./...                                     # unit
go test -v ./test/integration/... -timeout 10m    # integration (needs Docker)
```

Tests shelling to `ffmpeg`/`ffprobe` auto-skip when binaries missing — use `GenerateTestVideo` from `processor-steps/test_helpers.go`.

## Docs pointers — read these when the task calls for it

- **`docs/claude/features-index.md`** — read first when locating code. Maps every feature (worker lifecycle, queue, pipeline steps, MinIO ops, webhook, metrics) to file + canonical MinIO layout + queue names. Update when adding module, pipeline step, or new external contract.

- **`docs/claude/architecture.md`** — read before structural changes, adding pipeline step, or debugging e2e flow. Covers worker lifecycle, queue protocol (main / `:processing` / `:dead` / finished), job state machine, 7-step pipeline orchestration, resilience layers (circuit breakers, orphan recovery, retries).

- **`docs/claude/conventions.md`** — read before writing/modifying Go code. Rules: logging (zerolog, structured fields), error wrapping, critical vs non-critical step classification, config loading, metrics/tracing, circuit-breaker wrapping, test layout.

- **`docs/claude/design-decisions.md`** — read before proposing architectural change or questioning *why*. Explains `BRPOPLPUSH` over Streams, retry→DLQ policy, HLS single-command + fallback, NVENC auto-probe + CPU fallback, soft-archived raws with lifecycle rule, separate MinIO/Redis circuit breakers, webhook contract shape.

- **`docs/roadmap.md`** — read when user asks project status, remaining work, or what to build next. ~95% prod-ready; remaining work: scalability + Grafana dashboards.

- **`docs/GETTING_STARTED.md`**, **`docs/OBSERVABILITY.md`**, **`docs/TESTING.md`** — guides for setup, observability stack, full integration suite. Read when question is about operating worker, not code.

## Keeping this index healthy

- Add feature → update `features-index.md`.
- Change architecture/cross-cutting flow → update `architecture.md` (+ `design-decisions.md` if *why* changes).
- Change coding rule → update `conventions.md`.
- This file changes only when project structure changes.

## Padrão de mensagem de commit

Conventional Commits, **em português**, só o assunto — sem corpo, sem escopo, sem rodapé (nada de `Co-authored-by`).

Formato: `<tipo>: <verbo no infinitivo> <complemento>` — minúsculo depois do tipo, sem ponto final, até ~72 chars.

Tipos usados no repo (frequência real): `feat` > `chore` > `fix` > `refactor` > `docs` / `test`.

- `feat` — funcionalidade nova ou ampliada
- `fix` — correção de bug/comportamento
- `chore` — docs, README, migrations, scaffold, reorganização sem lógica
- `refactor` — renomear/reestruturar sem mudar comportamento
- `docs` / `test` — quando a mudança é só documentação ou só teste

Exemplos do histórico: `feat: adicionar upload de avatar do canal`, `fix: corrigir botão de reações`, `chore: atualizar README`, `refactor: renomear projeto`.

Título de PR (squash merge): `Feature/nome-da-branch (#N)`.

### Onde commitar

- **Padrão: direto na `master`.** Coisa pequena e bugfix não abre branch.
- **Exceção: feature grande** (vários commits). Aí **pergunte ao usuário** se é para criar `feature/<topic>` ou mandar direto para `master` — nunca decida sozinho.
- `master` sempre deployável; produção sai de tags `vX.Y.Z`.
