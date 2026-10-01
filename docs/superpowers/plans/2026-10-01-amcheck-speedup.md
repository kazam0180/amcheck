# AMCHECK Speedup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the AMCHECK pipeline keep up with 5,000+ (growing) apps by fixing failure bookkeeping first, then batching work, then raising sweep capacity.

**Architecture:** Keep the existing 4-stage shape (stats → matrix → run → aggregate) and the existing action pins. Change what flows through it: `ko-` failure markers written by the test loop itself, chunked matrix entries (10 apps per runner, one artifact per job), a single static-prefilter job, and a reusable workflow shared by both callers.

**Tech Stack:** GitHub Actions (workflow_call, matrix strategies, artifacts v7/v8 as already pinned), bash with `set -uo pipefail`, `comm`/`sort`/`grep -E`, `actions/checkout@v7`.

**Spec:** `docs/superpowers/specs/2026-10-01-amcheck-pipeline-spec.md` — the plan argues from the spec, so the spec travels with it; executors read both.

## Global Constraints

- Do not run anything resource-intensive on the local machine; heavy validation happens via canary `workflow_dispatch` runs, not local installs.
- Keep existing action pins (`actions/checkout@v7`, `actions/upload-artifact@v7`, `actions/download-artifact@v8`); do not upgrade or change pins in this plan.
- One task per commit to `main` (this repo's convention is direct pushes by automation); run the canary dispatch and read its results commit before starting the next task.
- Never `git add` anything under `results/` except `tested`, `tested.prev`, `excluded`, `log` (from Task 7 on; before that, do not make the stray-file problem worse).
- Matrix entry count must stay under the 256-entry GitHub limit (chunking is what makes this possible).

## Review Focus

- An app whose runner dies mid-batch (infra failure) leaves no `out-`/`ko-` marker and is retried next cycle forever without ever landing in `excluded`; a reasonable person expects retries, not silent exclusion — Task 2's test pins the retry behavior.
- A chunking off-by-one that drops or duplicates an app across chunks; a reasonable person expects the union of chunks to equal the selected list exactly — Task 2's test pins it.
- `comm` given unsorted input silently returns wrong sets; a reasonable person expects a loud failure — Task 1's `--check-order` test pins it.
- Two sweeps pushing `results/` concurrently silently losing one sweep's commit; a reasonable person expects a failed run, not lost data — Task 5's lease test pins it.
- A batch job killed by `timeout-minutes` mid-chunk losing completed apps' results; a reasonable person expects partial progress to be kept — Task 4's `always()` upload test pins it.

---

## File map

- Modify: `.github/workflows/AMCHECK.yml` (Tasks 1, 2, 4, 5, 6, 7)
- Modify: `.github/workflows/AMCHECK-manual-and-upload-icons.yml` (Tasks 1, 8)
- Create: `.github/amcheck-blocklist.txt` (Task 6 — data file, one extended regex per line)
- Create: `.github/actions/setup-runner/action.yml` (Task 3 — composite setup action)
- Create: `.github/workflows/_amcheck-core.yml` (Task 8 — reusable workflow)
- Modify: `results/` content only via workflow runs (Tasks 1, 7 — no hand edits)

---

### Task 1: Record failures and fix list plumbing

**Files:**
- Modify: `.github/workflows/AMCHECK.yml` (generate-matrix script ~lines 160–229, run-actions uploads ~lines 464–494, update-results loop ~lines 524–557)

**Interfaces:**
- Consumes: nothing new.
- Produces: `ko-<app>` marker contract (empty file = failed), `out-` uploaded with `if: always()`, `comm --check-order` enforced, `results/excluded` path spelled correctly — all later tasks rely on these.

- [ ] **Step 1: Write the failing check (static assertion script)**

Create `/tmp/ko-contract-check.sh` (throwaway probe, not committed):

```bash
#!/bin/bash
# Fails if any workflow references ko- artifacts without creating them,
# or references the misspelled result-var path.
fail=0
for f in .github/workflows/AMCHECK.yml; do
  grep -q 'results/ko-' "$f" || { echo "FAIL: $f never creates ko- markers"; fail=1; }
  grep -q 'result-var/excluded' "$f" && { echo "FAIL: $f contains result-var typo"; fail=1; }
  grep -q 'comm --check-order' "$f" || { echo "FAIL: $f comm lacks --check-order"; fail=1; }
done
exit $fail
```

Run: `bash /tmp/ko-contract-check.sh`
Expected: FAIL (all three conditions trip on current code).

- [ ] **Step 2: Fix `appslist` ordering and `comm` robustness**

In `show-stats` "Count programs" step, change:

```bash
sort programs/x86_64-apps | grep -v "\"kdegames\"\|\"kdeutils\"\|\"node\"\|\"platform-tools\"\| ffwa-\|am-utils" | awk '{print $2}' > appslist
```

to:

```bash
grep -v "\"kdegames\"\|\"kdeutils\"\|\"node\"\|\"platform-tools\"\| ffwa-\|am-utils" programs/x86_64-apps | awk '{print $2}' | sort -u > appslist
```

In `generate-matrix`, change both `comm -23` invocations to `comm --check-order -23`
(lines ~169 and ~174). Change `if [ -f result-var/excluded ]` to
`if [ -f results/excluded ]` (line ~193).

- [ ] **Step 3: Write `ko-` markers in the test step and always upload `out-`**

In the `test` step's failure branches, after each line that writes
`results/log-${{ matrix.file }}`, add a marker write. Concretely, change:

```bash
echo "${{ matrix.file }}" >> results/log-${{ matrix.file }}
exit 1
```

(LASTDIR=am branch, line ~414) to:

```bash
echo "${{ matrix.file }}" >> results/log-${{ matrix.file }}
echo "${{ matrix.file }}" >> results/ko-${{ matrix.file }}
exit 1
```

and change the `am -R` fallback (line ~456):

```bash
am -R "$LASTDIR" && echo "${{ matrix.file }}" >> results/ok-${{ matrix.file }} || echo "${{ matrix.file }}" >> results/log-${{ matrix.file }}
```

to:

```bash
am -R "$LASTDIR" && echo "${{ matrix.file }}" >> results/ok-${{ matrix.file }} || { echo "${{ matrix.file }}" >> results/log-${{ matrix.file }}; echo "${{ matrix.file }}" >> results/ko-${{ matrix.file }}; }
```

Change "Upload out" (line ~464) to add `if: always()`. Leave "Upload KO Results"
(`if: failure()`) and "Upload OK Results" (`if: success()`) as they are — they now
actually have files to upload.

- [ ] **Step 4: Verify the contract check passes and YAML parses**

Run: `bash /tmp/ko-contract-check.sh`
Expected: PASS (exit 0).

Run: `python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/AMCHECK.yml')); yaml.safe_load(open('.github/workflows/AMCHECK-manual-and-upload-icons.yml')); print('YAML OK')"` — if PyYAML is missing, use `ruby -ryaml -e` or skip with a note; do not install anything.
Expected: `YAML OK` (or a noted skip).

Run if available: `command -v actionlint && actionlint .github/workflows/AMCHECK.yml` else note the skip.
Expected: no findings, or skipped-with-note.

- [ ] **Step 5: Commit**

```bash
git add .github/workflows/AMCHECK.yml docs/superpowers/specs/2026-10-01-amcheck-pipeline-spec.md docs/superpowers/plans/2026-10-01-amcheck-speedup.md
git commit -m "fix(amcheck): record ko- failures, always upload out-, sort appslist after awk"
```

- [ ] **Step 6: Canary on main**

Dispatch `AMCHECK` with `what_test: "0ad 7zip"`, `retest_excluded: false`. After the
run, confirm a `results/tested` commit exists and both apps appear in it (or in
`results/excluded` with a real cause — either is a correct trace; silence is not).
Do not start Task 2 until this trace is read.

### Task 2: Batch 10 apps per runner, one artifact per job

**Files:**
- Modify: `.github/workflows/AMCHECK.yml` (generate-matrix output construction ~lines 217–229, whole `run-actions` job ~lines 231–494, update-results download/process ~lines 507–557)

**Interfaces:**
- Consumes: `ko-`/`out-` contract and `results/excluded` path from Task 1.
- Produces: chunked matrix entries `{"chunk": "<i>", "files": "<space-separated apps>"}`; per-job directory `results/job-<i>/`; single artifact `results-chunk-<i>` uploaded with `if: always()`; update-results loop over `results/job-*/out-*` with per-directory `ko-` lookup. Tasks 4–6 build on this loop.

- [ ] **Step 1: Write the failing chunking test (local fixture)**

Create `/tmp/chunk-test.sh` (throwaway probe, mirrors the exact loop to be added):

```bash
#!/bin/bash
# Fixture: 23 apps, CHUNK_SIZE=10 -> chunks of 10/10/3, union == input, no dupes.
seq -f "app%02.0f" 1 23 > /tmp/totest_selected.list
CHUNK_SIZE=10
i=0; n=0; chunk_files=""
MATRIX='{"include": ['
while IFS= read -r app || [ -n "$app" ]; do
  [ -z "$app" ] && continue
  if [ -z "$chunk_files" ]; then chunk_files="$app"; else chunk_files="$chunk_files $app"; fi
  n=$((n+1))
  if [ "$n" -ge "$CHUNK_SIZE" ]; then
    MATRIX+="{\"chunk\": \"$i\", \"files\": \"$chunk_files\"},"
    i=$((i+1)); n=0; chunk_files=""
  fi
done < /tmp/totest_selected.list
if [ -n "$chunk_files" ]; then MATRIX+="{\"chunk\": \"$i\", \"files\": \"$chunk_files\"},"; fi
MATRIX="${MATRIX%,}]}"
echo "$MATRIX" | grep -o '"files": "[^"]*"' | sed 's/"files": "//;s/"//' | tr ' ' '\n' | sort > /tmp/chunk_union.txt
sort /tmp/totest_selected.list > /tmp/chunk_expect.txt
diff /tmp/chunk_expect.txt /tmp/chunk_union.txt && echo "UNION OK"
echo "$MATRIX" | grep -o '"files": "[^"]*"' | sed 's/"files": "//;s/"//' | tr ' ' '\n' | sort | uniq -d | grep -q . && echo "DUPES FOUND" || echo "NO DUPES"
```

Run: `bash /tmp/chunk-test.sh`
Expected: `UNION OK` and `NO DUPES`. (This test passes standalone; it becomes the
regression gate for the workflow edit in Step 3 — if the pasted loop differs, rerun
this file with the pasted body.)

- [ ] **Step 2: Emit chunks from generate-matrix**

Replace the `FILES=$(head ...)` + `MATRIX` for-loop (lines ~217–229) with:

```bash
TOTEST_LIST=$(mktemp)
head -n "$how_many" totest.list > "$TOTEST_LIST" || cp totest.list "$TOTEST_LIST"
```

for the `what_test`/`retest_excluded` branches, write `$FILES` (or file contents) to
`$TOTEST_LIST` instead of using them directly. Then build chunks with the exact loop
from Step 1 (with `CHUNK_SIZE="${CHUNK_SIZE:-10}"`, reading `$TOTEST_LIST`), writing
`matrix=$MATRIX` to `$GITHUB_OUTPUT` as before. Keep the `skip=true` early exits
unchanged. Add `CHUNK_SIZE: 10` to the workflow `env` block (line ~46).

- [ ] **Step 3: Loop over apps inside run-actions**

Change the matrix to `matrix: ${{ fromJson(needs.generate-matrix.outputs.matrix) }}`
(unchanged shape, new entry fields). Set job `timeout-minutes: 360` (revisited with
real DURATION data in Task 4). Replace per-app steps ("APP", "Is AppImage?",
"Is on GitHub?", "SITE", "test") with a single step:

```bash
CHUNK="${{ matrix.chunk }}"
FILES="${{ matrix.files }}"
mkdir -p "results/job-$CHUNK"
for app in $FILES; do
  out="results/job-$CHUNK/out-$app"
  echo "APP=\"$app\"" >> "$out"
  if grep -qe "appimage-extract\|mage\$\|tmp/\*mage" "$app" 1>/dev/null; then
    echo "APPIMAGE='yes'" >> "$out"
  else
    echo "APPIMAGE='no'" >> "$out"
  fi
  if grep -q "api.github.com" "$app" 2>/dev/null; then
    echo "GITHUB='yes'" >> "$out"
  else
    echo "GITHUB='no'" >> "$out"
  fi
  if grep -q "^SITE=" "$app" 2>/dev/null; then
    SITE=$(eval echo "$(grep -i '^SITE=' "$app" | head -1 | sed 's/SITE=//g')")
    echo "SITE=\"$SITE\"" >> "$out"
  fi
  <existing install/verify body with ${{ matrix.file }} replaced by $app,
   out/log/ok/ko paths prefixed with results/job-$CHUNK/,
   plus start=$(date +%s) before install and
   echo "DURATION=$(( $(date +%s) - start ))" >> "$out" after,
   and `continue` past a failed app instead of exiting the whole job>
done
```

Working directory stays `programs/x86_64`. Replace the three upload steps with one:

```yaml
- name: "Upload chunk results"
  if: always()
  uses: actions/upload-artifact@v7
  with:
    name: results-chunk-${{ matrix.chunk }}
    path: programs/x86_64/results/job-${{ matrix.chunk }}
    if-no-files-found: error
    retention-days: 1
    compression-level: 0
```

- [ ] **Step 4: Aggregate job directories in update-results**

After the download step, loop `for LogFile in results/job-*/out-*;` (instead of
`results/out-*`), derive `dir=$(dirname "$LogFile")`, and check
`KoFile="$dir/ko-$appname"` (instead of `results/ko-$appname`). After the loop and
`git add results`, add `rm -rf results/job-*` so job directories are not committed.
Keep the tested/excluded/log updates byte-identical otherwise.

- [ ] **Step 5: Verify**

Run: `bash /tmp/chunk-test.sh` (regression) — Expected: `UNION OK`, `NO DUPES`.
Run: YAML parse as in Task 1 Step 4; `bash -n` on the new loop body saved to a
temp file; `actionlint` if available. Expected: clean or noted skips.
Count upload steps: `grep -c "upload-artifact" .github/workflows/AMCHECK.yml`
should show exactly 2 remaining (`appslist` + chunk upload).

- [ ] **Step 6: Commit and canary**

```bash
git add .github/workflows/AMCHECK.yml
git commit -m "perf(amcheck): 10 apps per runner, one artifact per chunk job"
```

Dispatch canary with `what_test: "0ad 7zip"`. Confirm: ≤2 chunk jobs ran, one
`results-chunk-*` artifact per job, and both apps traced in `tested`/`excluded`.
Confirm no `job-*` directories were committed.

### Task 3: Define provisioning once (composite setup action)

**Files:**
- Create: `.github/actions/setup-runner/action.yml`
- Modify: `.github/workflows/AMCHECK.yml` (replace "Cache dependencies", "Fix AppArmor", "Install AM manually" steps with one `uses:`)

**Interfaces:**
- Consumes: chunk loop from Task 2 (must keep working identically).
- Produces: `setup-runner` composite action with no inputs and one output
  `am_version` (first line of `am --version`, for log enrichment). Task 8 reuses it.

- [ ] **Step 1: Write the action with the exact current behavior**

```yaml
name: "Setup AMCHECK runner"
description: "Install apt deps, relax AppArmor, install AM manually (moved verbatim from AMCHECK run-actions)"
outputs:
  am_version:
    description: "First line of am --version"
    value: ${{ steps.amver.outputs.am_version }}
runs:
  using: "composite"
  steps:
    - name: "Cache dependencies"
      uses: awalsh128/cache-apt-pkgs-action@latest
      with:
        packages: curl libnotify-bin ripgrep wget
        version: 1.0
    - name: "Fix AppArmor"
      shell: bash
      run: |
        sudo sysctl -w kernel.apparmor_restrict_unprivileged_unconfined=0
        sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
    - name: "Install AM manually"
      shell: bash
      run: |
        sudo mkdir -p /opt/am/modules /usr/local/bin || exit 1
        sudo cp -r ./APP-MANAGER /opt/am/APP-MANAGER && sudo chmod a+x /opt/am/APP-MANAGER || exit 1
        sudo ln -fs /opt/am/APP-MANAGER /usr/local/bin/am || exit 1
        sudo touch /opt/am/remove || exit 1
        sudo cp -r modules/*.am /opt/am/modules/
        export AMDATADIR="${XDG_DATA_HOME:-$HOME/.local/share}"/AM
        mkdir -p "$AMDATADIR" || exit 1
        cp -r programs/x86_64-apps "$AMDATADIR"/ || exit 1
    - name: "Record AM version"
      id: amver
      shell: bash
      run: echo "am_version=$(am --version 2>/dev/null | head -1)" >> "$GITHUB_OUTPUT"
```

- [ ] **Step 2: Use it in run-actions**

Replace the three steps with:

```yaml
- name: "Setup runner"
  uses: ./../actions/setup-runner
```

with the path adjusted to the checkout layout at that point in the job (the AM
checkout is at the job root; the `results` repo checkout or this repo must provide
the action — since run-actions checks out `AM`, not this repo, add a prior step
checking out this repo to `path: amcheck` if not already present, and use
`./amcheck/.github/actions/setup-runner`). Verify the relative path against the
actual job step order while editing; the committed file must contain the verified path.

- [ ] **Step 3: Verify**

YAML parse + `actionlint` if available. Confirm no behavior change: diff the action's
`run:` bodies against the removed steps — they must be byte-identical except `shell:`.
Canary dispatch `what_test: "0ad 7zip"`; confirm install works through the action.

- [ ] **Step 4: Commit**

```bash
git add .github/actions/setup-runner/action.yml .github/workflows/AMCHECK.yml
git commit -m "refactor(amcheck): share runner provisioning via composite action"
```

### Task 4: Bound per-app time and keep partial progress

**Files:**
- Modify: `.github/workflows/AMCHECK.yml` (install/verify loop body from Task 2, job `timeout-minutes`, `env`)

**Interfaces:**
- Consumes: chunk loop (Task 2), setup action (Task 3).
- Produces: bounded installs (2 attempts, `TIMEOUT` env wired as minutes), per-app
  `DURATION=` in every `out-` file, partial-chunk uploads preserved on timeout.

- [ ] **Step 1: State the failing bound**

Current worst case per app: 3 × `timeout 1800` + 2 × `sleep 30` ≈ 91 minutes, and
`env.TIMEOUT: 23` is never referenced. Write the bound as code in the loop.

- [ ] **Step 2: Implement the bound**

Set `env.TIMEOUT: 23` meaning minutes; at the top of the loop step add:

```bash
TIMEOUT_S=$(( ${TIMEOUT:-23} * 60 ))
```

Replace every `timeout 1800 am -i ...` with `timeout "$TIMEOUT_S" am -i ...`.
In each of the three install branches (interactive `read`/`wine`, `bat-extras`,
default), delete the **third** attempt block (the second `sleep 30` + retry),
leaving initial attempt + one retry with `sleep 10` between them.
The chunk upload step already has `if: always()` from Task 2 — confirm it is still
present (this is the Review Focus #5 pin: a job killed by `timeout-minutes` must
still upload completed apps' markers).

- [ ] **Step 3: Verify**

`bash -n` on the loop body; `grep -c "timeout 1800"` must return 0;
`grep -c 'timeout "$TIMEOUT_S"'` must equal the number of install call sites (3);
canary `what_test: "0ad 7zip"`; read both `out-` contents from the downloaded
artifacts (via the results commit's `log` lines) and confirm `DURATION=` lines exist.

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/AMCHECK.yml
git commit -m "perf(amcheck): 2 install attempts, wire TIMEOUT minutes, log DURATION"
```

- [ ] **Step 5: Tune from data (same task, after one full sweep)**

After the next full hourly run, read the committed `results/log` DURATION lines,
take the p95, and set chunk-job `timeout-minutes` to `CHUNK_SIZE × p95 × 2 attempts
+ 15 min setup headroom`, rounded up. Commit the value with the p95 figure in the
message. Do not proceed to Task 5 until this number exists.

### Task 5: Raise sweep capacity past 250/hour

**Files:**
- Modify: `.github/workflows/AMCHECK.yml` (`env.TOTEST`, `on.schedule`, `on.push`, concurrency comment)

**Interfaces:**
- Consumes: bounded chunks (Task 4) with measured p95 and tuned `timeout-minutes`.
- Produces: `TOTEST: 500` hourly sweeps; documented gate for 30-minute cadence;
  dead `push` trigger removed; `what_test` dispatch documented as the changed-apps
  fast path. Produces the push-lease behavior Task 7's rotation relies on (unchanged
  `push --force-with-lease`).

- [ ] **Step 1: Prove the lease fails loudly (no silent clobber)**

Read the "Push" step: it uses `git push --force-with-lease` and prints
`git diff && git status` on failure. Add `exit 1` to that failure branch so
overlapping runs fail visibly instead of exiting 0:

```bash
git push --force-with-lease && echo "sync successfull" >> $GITHUB_STEP_SUMMARY || (git diff && git status && exit 1)
```

This is the Review Focus #4 pin.

- [ ] **Step 2: Raise TOTEST to 500**

Change `env.TOTEST: 250` to `500`. With `CHUNK_SIZE=10` this yields ~50 matrix
entries — well under the 256 cap, which is the entire point of chunking. Keep
`how_many` input max note (`max 256`) as-is for manual runs, or raise its
description only if the 256 matrix-entry cap is respected (50 entries « 256, so a
manual `how_many: 500` is safe; update the input description to `(max 500)`).

- [ ] **Step 3: Remove the dead push trigger, document the fast path**

Delete the `push: branches: main, paths: programs/x86_64/**` block (this repo has
no `programs/` directory; it never fires). Append a comment above
`workflow_dispatch` inputs:

```yaml
# Fast path for changed apps: repository_dispatch (or manual dispatch) with
# what_test set to the changed app names tests only those apps; the hourly
# sweep covers the remainder. There is no push trigger in this repo because
# install scripts live in the AM repo, not here.
```

- [ ] **Step 4: Gate the 30-minute schedule (do NOT enable yet)**

Add, commented out, below the current schedule with the enabling condition:

```yaml
schedule:
  - cron: '0 * * * *'
  # Enable ONLY after a full TOTEST=500 run completes in under 25 minutes wall-clock
  # (read from run duration in the Actions UI) AND owner accepts ~2x runner minutes:
  # - cron: '*/30 * * * *'
```

- [ ] **Step 5: Verify, commit, canary**

YAML parse + `actionlint` if available. Commit:

```bash
git add .github/workflows/AMCHECK.yml
git commit -m "feat(amcheck): sweep 500 apps/hour, fail loudly on push race, drop dead trigger"
```

Canary `what_test: "0ad 7zip"`; then watch one full hourly run: confirm ~50 chunk
jobs, run wall-clock, and a clean `results` commit. Record the wall-clock time —
it is the input to Task 4's tuning (if not done) and the 30-minute gate.

### Task 6: Single static-prefilter job, same-round skip

**Files:**
- Create: `.github/amcheck-blocklist.txt`
- Modify: `.github/workflows/AMCHECK.yml` (new `static-checks` job, slimmer batch loop, `run-actions` needs + static download)

**Interfaces:**
- Consumes: chunk loop (Task 2), `ko-` contract (Task 1).
- Produces: `static-checks` job + `static` artifact (`static/out-<app>`,
  `static/ko-<app>` for blocklisted); matrix loop skips install for apps with
  `static/ko-<app>` and reuses `static/out-<app>` metadata lines.

- [ ] **Step 1: Create the blocklist from the hardcoded lines**

Create `.github/amcheck-blocklist.txt` with one extended regex per line, transcribed
from the current hardcoded branches:

```regex
animashooter-junior|animashooter-pioneer|kiwix|ryujinx|ryujinx-canary
mudlet|openxcom|openxcom-extended|vhc-viewer-wayland|vhc-viewer-x11
koreader-nightly
^(node|npm)$
wine
bat-extras
```

Keep one pattern per line (drop the `^...$`/grouping into plain `grep -E -f`
compatible lines as above). Add a header comment line starting with `#` and make
the matching step strip `#` lines (`grep -v '^#'`).

- [ ] **Step 2: Add the static-checks job**

New job after `generate-matrix`, one runner, no AM install — only the AM checkout
(for program files) and the `totest` list. `generate-matrix` must additionally
upload the selected list: add an upload of `$TOTEST_LIST` as artifact `totest`
(retention 1 day). The job:

```yaml
static-checks:
  name: "static 🔍"
  needs: generate-matrix
  if: ${{ needs.generate-matrix.outputs.skip != 'true' }}
  runs-on: ubuntu-latest
  steps:
    - name: "Checkout AM"
      uses: actions/checkout@v7
      with:
        repository: ${{ env.REPO }}
    - name: "Download totest"
      uses: actions/download-artifact@v8
      with:
        name: totest
    - name: "Static checks"
      run: |
        mkdir -p static
        grep -v '^#' .github/amcheck-blocklist.txt > /tmp/blocklist || true
        while IFS= read -r app || [ -n "$app" ]; do
          [ -z "$app" ] && continue
          out="static/out-$app"
          echo "APP=\"$app\"" >> "$out"
          if grep -qe "appimage-extract\|mage\$\|tmp/\*mage" "$app" 1>/dev/null; then
            echo "APPIMAGE='yes'" >> "$out"
          else
            echo "APPIMAGE='no'" >> "$out"
          fi
          if grep -q "api.github.com" "$app" 2>/dev/null; then
            echo "GITHUB='yes'" >> "$out"
          else
            echo "GITHUB='no'" >> "$out"
          fi
          if grep -q "^SITE=" "$app" 2>/dev/null; then
            SITE=$(eval echo "$(grep -i '^SITE=' "$app" | head -1 | sed 's/SITE=//g')")
            echo "SITE=\"$SITE\"" >> "$out"
          fi
          if echo "$app" | grep -Eqf /tmp/blocklist; then
            echo "$app: blocklisted, install skipped" >> "$out"
            echo "$app" >> "static/ko-$app"
          fi
        done < totest_selected.list
```

Note: this job needs this repo's blocklist file too — check out this repo (or fetch
the single file) alongside AM. Upload artifact `static`, path `static/`.

- [ ] **Step 3: Slim the batch loop and skip blocked same-round**

In `run-actions`, add `static-checks` to `needs`, download the `static` artifact,
and at the top of the per-app loop add:

```bash
if [ -f "static/ko-$app" ]; then
  cat "static/out-$app" >> "$out"
  echo "$app" >> "results/job-$CHUNK/ko-$app"
  continue
fi
```

and remove the now-duplicated APPIMAGE/GITHUB/SITE grep block from the loop,
replacing it with `cat "static/out-$app" >> "$out"`. Delete the hardcoded
blacklist branches (they now live in the blocklist file). `update-results` is
unchanged — `ko-` markers flow into `excluded` via Task 1 logic.

- [ ] **Step 4: Verify, commit, canary**

Fixture test: `echo koreader-nightly | grep -Eqf .github/amcheck-blocklist.txt`
must match; `echo 0ad | grep -Eqf ...` must not. YAML parse + `actionlint` if
available. Commit:

```bash
git add .github/amcheck-blocklist.txt .github/workflows/AMCHECK.yml
git commit -m "perf(amcheck): single static-prefilter job, blocklist file, same-round skip"
```

Canary with a known-blocked app plus a good one
(`what_test: "koreader-nightly 0ad"`): confirm `koreader-nightly` lands in
`excluded` without an install attempt (no DURATION line), `0ad` installs normally.

### Task 7: Stop bloating the results repo; rotate instead of wiping

**Files:**
- Modify: `.github/workflows/AMCHECK.yml` (`update-results` checkout + cleanup + push; `already-tested` job)

**Interfaces:**
- Consumes: job-dir aggregation (Task 2), lease-fails-loudly (Task 5).
- Produces: `results/` commits containing only `tested`, `tested.prev`, `excluded`,
  `log`; one-time removal of the ~503 stray files; cycle rotation preserving the
  previous full list.

- [ ] **Step 1: One-time removal of stray committed files**

On a local clone (read-only inspection first: `git ls-files results/ | grep -v -e
'tested$' -e 'excluded$' -e '/log$'` to list them — this lists names only, no heavy
transfer beyond the clone you already have), then:

```bash
git ls-files results/ | grep -v -e '/tested$' -e '/excluded$' -e '/log$' | xargs git rm
git commit -m "chore(results): drop stray per-app binaries, keep tested/excluded/log"
git push
```

If the stray count is not ~500 as expected, stop and report instead of pushing.

- [ ] **Step 2: Clean before every commit in update-results**

After the process-log loop and before `git add results`, insert:

```bash
rm -rf results/job-* results/icons results/out-* results/ok-* results/ko-*
git add results
git status --porcelain | grep -v -e 'results/tested' -e 'results/tested.prev' -e 'results/excluded' -e 'results/log' | grep '^A ' && { echo "unexpected new tracked file"; exit 1; } || true
```

(The guard fails the run if anything outside the four tracked files would be
newly added. Pre-existing stray files were removed in Step 1.)

- [ ] **Step 3: Replace the wipe with rotation**

Replace the `already-tested` job's "Reset all tested" run block:

```bash
git rm -r results
git commit -m "reset stats"
git push --force
git clean -f
```

with:

```bash
mv results/tested results/tested.prev
git add results/tested.prev
rm -f results/tested
git add results/tested || true
git commit -m "cycle complete: rotate tested to tested.prev"
git push --force-with-lease
```

`excluded` and `log` are kept (never wiped). Next cycle retests everything while
`tested.prev` preserves the previous signal; the following completion overwrites it.
Do not change the `skip == 'true'` condition.

- [ ] **Step 4: Verify, commit, canary**

YAML parse + `actionlint` if available. Commit:

```bash
git add .github/workflows/AMCHECK.yml
git commit -m "fix(amcheck): keep results/ to four tracked files, rotate cycles instead of wiping"
```

Canary `what_test: "0ad 7zip"`; confirm the results commit contains no `job-*`,
`out-*`, `ok-*`, or icon files (`git show --stat` on the bot commit).

### Task 8: Deduplicate the two workflows via a reusable workflow

**Files:**
- Create: `.github/workflows/_amcheck-core.yml`
- Modify: `.github/workflows/AMCHECK.yml` (becomes triggers + caller)
- Modify: `.github/workflows/AMCHECK-manual-and-upload-icons.yml` (becomes triggers + caller)

**Interfaces:**
- Consumes: all shared jobs in their post-Task-1–7 shape.
- Produces: `_amcheck-core.yml` (`on: workflow_call`, inputs `totest` (number,
  default 500), `chunk_size` (number, default 10), `upload_icons` (boolean, default
  false)); both callers reduced to `on:` triggers + one `uses: ./_amcheck-core.yml`
  job with `secrets: inherit`.

- [ ] **Step 1: Extract verbatim**

Move `show-stats`, `generate-matrix`, `static-checks`, `run-actions`,
`update-results`, `already-tested` jobs byte-identical into `_amcheck-core.yml`,
replacing `env.TOTEST` references with `${{ inputs.totest }}` and `CHUNK_SIZE`
default with `${{ inputs.chunk_size }}`. Schedules cannot live in a called
workflow — keep `schedule`, `repository_dispatch`, `workflow_dispatch`, and the
`concurrency` group in each caller. `upload_icons` gates the icons-upload step
(already present in the manual file, absent/commented in AMCHECK — unify on the
manual file's enabled form behind the flag).

- [ ] **Step 2: Write the thin callers**

`AMCHECK.yml` becomes: name, `on` (schedule hourly + dispatch, no push trigger per
Task 5), `concurrency`, `permissions`, one job calling the core with
`totest: 500, chunk_size: 10, upload_icons: false`. Manual file: same with its
dispatch inputs passed through (`what_test`, `how_many`, `reset_stats`,
`retest_excluded`) and `upload_icons: true`.

- [ ] **Step 3: Verify both callers**

YAML parse all three files + `actionlint` each if available. Confirm the core file
contains no `on.schedule`/`on.push` (grep must show only `workflow_call`), and each
caller contains no job logic beyond the call. Dispatch canary on **both**
workflows (`what_test: "0ad 7zip"`); both must trace results. This is the only
task that touches the manual workflow's behavior — the icon step runs only when
`upload_icons: true`.

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/
git commit -m "refactor(amcheck): share pipeline via reusable workflow"
```

## Self-review (run by the plan author, done)

1. **Spec coverage:** R1→Task 1, R2→Task 2, R3→Task 3, R4→Task 4, R5→Task 5, R6→Task 6, R7→Task 7, R8→Task 8. All covered.
2. **Placeholder scan:** no TBD/TODO; every code step shows exact YAML/bash; verification commands are exact, with `actionlint` explicitly optional-gated on availability.
3. **Type consistency:** artifact names (`results-chunk-<i>`, `static`, `totest`), marker names (`out-/ok-/ko-`), matrix fields (`chunk`, `files`), and the four tracked result files are spelled identically in every task.
4. **Review Focus:** all five lines have owning tests — #1 Task 2 Step 1/6 (retry trace), #2 Task 2 Step 1 (union/dupes), #3 Task 1 Steps 1–2 (`--check-order`), #4 Task 5 Step 1 (lease `exit 1`), #5 Task 2 Step 3 + Task 4 Step 2 (`if: always()` confirmation).
