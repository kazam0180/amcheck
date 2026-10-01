# AMCHECK Pipeline Spec

Date: 2026-10-01. Source: static read of `.github/workflows/AMCHECK.yml` (634 lines),
`.github/workflows/AMCHECK-manual-and-upload-icons.yml` (608 lines), `results/tested`
(3,948 lines), `results/log` (19,739 lines). No workflow was executed; all claims are
read, not observed.

## 1. What the pipeline does

Four serial stages in `AMCHECK.yml`:

1. `show-stats` — clones `ivan-hc/AM`, builds `appslist` (line 82), uploads it as an
   artifact; counts lines in `results/tested` / `results/excluded`.
2. `generate-matrix` — `comm -23 appslist results/tested`, minus `results/excluded`,
   `head -n 250` (`TOTEST`, line 49) → job matrix with **one entry per app**.
3. `run-actions` — one `ubuntu-latest` runner per app: checkout AM, apt-install deps,
   manual AM install, `am -i <app>` with `timeout 1800` retried up to 3× with
   `sleep 30` (lines 367–406), verify `/opt/<app>/remove`, `am -R`. Uploads `out-<app>`
   always, `ok-<app>` on success.
4. `update-results` — downloads all artifacts merged, loops over `out-*`, appends to
   `results/tested` / `results/excluded` / `results/log`, commits, pushes.

Triggers: hourly cron `0 * * * *` (AMCHECK only), `repository_dispatch`, and
`workflow_dispatch` with inputs `reset_stats`, `retest_excluded`, `what_test`,
`how_many`. The `push` trigger on `programs/x86_64/**` is dead: this repo contains no
`programs/` directory.

## 2. Result-file and artifact contracts (current)

- `results/tested`: app names, one per line, `sort -u` maintained.
- `results/excluded`: same format. **Currently absent from the repo** (nothing writes it).
- `results/log`: 5 lines per app (`APP=`, `APPIMAGE=`, `GITHUB=`, `SITE=`, separator).
- Per-app artifacts: `out-<app>` (metadata, uploaded on success only — the upload step
  has no `if:`, i.e. default `success()`), `ok-<app>` (uploaded `if: success()`),
  `ko-<app>` (uploaded `if: failure()`, **but nothing ever creates this file** —
  verified by full grep of both workflows).
- `results/` also contains ~503 stray per-app files (icons, `404: Not Found` bodies,
  e.g. `results/clarity`, `results/zero-limit`) committed accidentally via
  `git add results` after artifact download with `merge-multiple: true`.
- `already-tested` job: when `appslist == results/tested`, runs `git rm -r results`
  and force-pushes — wiping all coverage signal instead of rotating it.

## 3. Target contracts (what the plan implements)

- `ko-<app>`: created by the test loop itself on any install/verify failure (empty
  marker file; human-readable reason goes to `log-<app>` lines merged into `results/log`).
  Presence of `ko-<app>` is the **sole** failure signal; job exit codes carry no signal.
- `out-<app>`: uploaded with `if: always()` so every attempt leaves a trace.
- `results/excluded`: grows via `ko-` markers; never wiped by rotation.
- Chunked matrix entry: `{"chunk": "<i>", "files": "<app1> <app2> ..."}` with
  `CHUNK_SIZE=10`; one artifact `results-chunk-<i>` per job containing
  `job-<i>/out-*`, `job-<i>/ok-*`, `job-<i>/ko-*`; uploaded with `if: always()`.
- `results/tested.prev`: previous full-cycle list, kept for reference across rotation.
- `results/` tracked files going forward: `tested`, `tested.prev`, `excluded`, `log`
  only. Everything else downloaded is deleted before `git add`.
- `.github/amcheck-blocklist.txt`: one extended regex per line, matched with
  `grep -E -f` against app names. A match means "do not install; record as excluded
  with reason".

## 4. Requirements (plan tasks in order)

- R1: Every install attempt ends in `tested` or `excluded`. No silent drops.
  Fix `comm` input ordering (`sort` after `awk`), add `comm --check-order`, fix the
  `result-var/excluded` typo (both workflows line ~193).
- R2: Run ≤ ~50 matrix jobs per sweep (not 250–500), one artifact per job.
- R3: Provisioning (checkout, apt, AM install) defined once and reused.
- R4: Per-app install bounded: 2 attempts, `TIMEOUT` env (23) wired as minutes,
  job-level `timeout-minutes`, per-app `DURATION=` logged to `out-` files.
- R5: Sweep capacity ≥ 500 apps/hour at hourly cadence; 30-minute cadence only after
  a measured full run completes in < 25 minutes; dead `push` trigger removed.
- R6: One `static-checks` job per sweep does all greps + blocklist matching;
  matrix jobs skip install for blocklisted apps same-round.
- R7: Shallow, small checkouts where lease semantics allow; no binaries committed;
  cycle completion rotates `tested → tested.prev` instead of wiping.
- R8: One reusable workflow holds the shared jobs; the two existing files become
  thin callers (triggers + parameters only).

## 5. Non-goals

No changes to the AM repo, to install scripts under test, or to what "correct install"
means. No new credentials, no self-hosted runners, no paid-minute commitments made by
this plan (R5's 30-minute option is gated on measurement and called out as a cost
decision for the owner).
