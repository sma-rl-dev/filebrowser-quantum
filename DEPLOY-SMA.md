# FileBrowser Quantum — tester-env deployment

Baseline: `v2.0.2-stable` @ `2c071d1e6eb48d4b88da2657885d345663a0a88b` (upstream
`gtsteffaniak/filebrowser`). Latest stable release at triage; the newer v2.1.x
line ships only as `*-beta` tags, so the stable tag is the pin. All build and
runtime tooling lives in this fork; no prebuilt app image is used.

## Quick Start

```bash
./tester-env deploy    # build from source, start container, wait for health+login
./tester-env seed      # install the seed data set, restart to index it
./tester-env verify    # health, login, SPA assets, admin, seed tree, users, share
./tester-env reset     # remove container + /data volume (keeps the image)
```

`--run-id <id>` isolates container/volume names for parallel runs, `--port`
changes the host port (default `18078`, container port `80`). `scripts/rl-env`
sets `IMAGE_TAG` to a content-addressed tag; manual runs use
`tester-env-filebrowser-quantum:dev`.

- URL: `http://localhost:18078` (browser origin `http://host.docker.internal:18078`)
- Credentials: `admin` / `tester-env-admin` (`ADMIN_PASSWORD` overrides; the
  container gets it as `FILEBROWSER_ADMIN_PASSWORD`)
- Container: `tester-env-filebrowser-quantum`, volume
  `filebrowser-quantum-data` (sqlite + files + cache)

## Build

`Dockerfile.tester-env` mirrors upstream `_docker/Dockerfile`:

1. `ghcr.io/gtsteffaniak/ffmpeg:9.0.2-decode` supplies `ffmpeg`/`ffprobe`;
2. `golang:1.27.4-alpine` builds with `CGO_ENABLED=1 -tags mupdf,musl`
   (`analytics` intentionally dropped) after `go mod download` and
   `go run ./scripts/setup-gofitz-cgo`; Go module and build caches are BuildKit
   cache mounts, and the module/header layers precede the source COPY;
3. `node:jod-slim` (Node 22 bookworm) runs `npm ci` + `npm run build:docker`
   (Vite → `backend/internal/web/dist`), with an npm cache mount and the
   lockfile layer before the source COPY;
4. `alpine:3.23` runtime copies the SPA build into `./internal/web/dist`, runs as
   uid 1000, healthchecks `/health`. Seed data is not baked into the image:
   `seed` writes it into the `/data` volume.

Every base image is pinned by tag + manifest digest. The SPA copy matters:
`backend/cmd/root.go` serves `internal/web/dist` from disk when it exists; the
`embed/` fallback is empty in this checkout and would render a blank UI.

Measured on this host (28 cores, Docker 28.5.1, BuildKit):

| Rebuild | Wall time |
| --- | --- |
| First build, empty caches | 99.3 s (npm ci ~44 s, Go build ~23 s, Vite ~24 s) |
| Frontend-only source change | 17.6 s (Vite 15.2 s, Go stage cached) |
| Go-only source change | 6.4 s (go build 4.6 s, Vite stage cached) |
| No-op | 1.2 s |

## Config and determinism

`tester-env.yaml` is baked to `/home/filebrowser/config/config.yaml`:

- `auth.adminUsername` / `auth.adminPassword` and `auth.key` are fixed, so the
  admin is recreated identically after every reset (no random bootstrap
  password, no per-boot JWT key).
- `server.disableUpdateCheck: true` removes the GitHub tag check.
- The binary is built without the `analytics` tag, so `publish_stub.go` compiles
  and nothing is POSTed to `api.filebrowserquantum.com`.
- `server.sources` is one source named `files` at `/data/files`;
  `server.database.path` is `/data/database.sqlite`; `server.cacheDir` is
  `/data/cache`. All three live on the named volume.

## Seed data

Theme: Halcyon Studio, a small design studio sharing one drive. `seed` stops the
container, wipes `/data/files`, the index (`/data/cache/sql`), thumbnails and the
sqlite database, writes the tree with fixed mtimes, creates the standard users
with the app CLI (`./filebrowser user set`) and starts the container again so the
boot scan indexes the tree. The admin is recreated from `config.yaml` on boot, so
`seed` is a full data install: re-seeding (or reset + deploy + seed) returns the
same visible state.

Tree: 24 files in 18 folders.

```text
README.md  team-roster.csv  office-floor-plan.png
01-Projects/        Atlas-Redesign/{brief.md,budget.csv,assets/{atlas-hero.png,atlas-logo.png}},
                    Beacon-Mobile-App/{release-checklist.md,api-endpoints.json,screenshots/beacon-login.png},
                    Cedar-Portal-Migration/scope.md, "Q2 Client Kickoff Notes.md"
02-Finance/         2026-Q1/{invoices.csv,budget-vs-actual.csv}, 2026-Q2/invoices.csv,
                    Vendor-Contracts/printing-services-contract.md
03-Operations/      Policies/{remote-work-policy.md,equipment-request-policy.md},
                    Meeting-Notes/{2026-08-14-studio-sync.md,2026-09-02-client-kickoff.md}
04-Marketing/       Campaign-Q3-Launch/{campaign-brief.md,channel-plan.csv}, Brand/brand-guidelines.md
Archive/            2025-Projects/legacy-site-refresh.md
```

Fixed mtimes spread over 2026-04-22 .. 2026-10-08 keep the Modified column and
date sorting stable. File sizes are small text/CSV/JSON/Markdown plus four tiny
PNGs (image previews active). One file name contains spaces.

Accounts: `admin` / `tester-env-admin` (administrator) and standard users
`ava.chen`, `noah.patel`, `mila.rossi` / `tester-env-user`. `team-roster.csv`
lists those three with their roles.

Share: one link titled `Atlas Redesign Client Review` for
`/01-Projects/Atlas-Redesign` (normal share, no expiry, anonymous view allowed).
Its hash is generated per creation, so assert the title/path, not the hash.

`verify` asserts `/health`, admin login, the SPA from `internal/web/dist`, the
exact root listing, the `/01-Projects` listing, 24 files / 18 folders on disk,
the four users, `ava.chen` login, the single share, and prints a fingerprint of
the tree (relative path + mtime + content). Two consecutive
reset -> deploy -> seed -> verify cycles produced
`7ba22b457335f7f3bd9b4ea0b3c15df9c3d9c14f691c005fe6c848ef95e9c08b`.

Reset removes the container and the volume; the image is kept because
content-addressed tags are shared across runs. Cleanup of unused images is
`scripts/rl-env gc --prune-images`.

## Mutation notes

- Frontend mutations rebuild only the Vite stage plus the runtime copies.
- Go mutations reuse the Vite stage and re-run only the Go build with a warm
  `GOCACHE`; the CGO MuPDF compile of go-fitz is the expensive part and stays
  cached.
- Mutating the embedded SPA (`frontend/`) is served from `internal/web/dist`;
  no need to touch `embed/`.
- `-trimpath -buildvcs=false` and build-arg version/commit keep the binary
  reproducible across pass-variant builds.

## Browser proof

Browser-verified 2026-11-19 at `http://host.docker.internal:18078/` after login
`admin` / `tester-env-admin`:

- Smoke: created a folder through the UI (New Folder action) and it appeared in
  the listing; the later reset cycle removed it (checked on disk).
- Seeded state: the root listing showed the five folders `01-Projects`,
  `02-Finance`, `03-Operations`, `04-Marketing`, `Archive` plus the files
  `README.md`, `office-floor-plan.png`, `team-roster.csv`; `/01-Projects` showed
  `Atlas-Redesign`, `Beacon-Mobile-App`, `Cedar-Portal-Migration` and
  `Q2 Client Kickoff Notes.md`; `01-Projects/Atlas-Redesign/brief.md` opened in
  the Markdown preview; Settings > Users listed `admin`, `ava.chen`,
  `mila.rossi`, `noah.patel`; Settings > Shares listed one
  `Atlas Redesign Client Review` share for `/01-Projects/Atlas-Redesign`.

Seed requirement ids for scenario checks live in the corpus at
`scenarios/filebrowser-quantum/seed.manifest.json` (the sibling `filebrowser`
fork keeps the manifest in the corpus too, so it is not duplicated here).

## Caveats

- Previews need the ffmpeg stage and the `mupdf` tag; dropping either removes
  PDF/office/media preview oracles.
- Share expiry countdowns, activity timestamps and index times are time-dependent
  and should not be asserted literally; seeded file mtimes are fixed.
- Users and access rules are CLI-creatable and shares are API-creatable
  (`./filebrowser user set ... -c /home/filebrowser/config/config.yaml`,
  `./filebrowser set rule ...`, `POST /api/share`). `seed` runs the CLI against a
  stopped container and the share against the started one.
- The app listens on `0.0.0.0:80` inside the container; no reverse proxy or TLS
  is involved.
