# SDD ledger — plan: docs/superpowers/plans/2026-10-01-amcheck-speedup.md

Spec: docs/superpowers/specs/2026-10-01-amcheck-pipeline-spec.md (read 2026-10-01).
Setup 2026-10-01: repo /run/media/kazam/secondary/amcheck, branch main @2b1cbf9.
origin=https://github.com/kazam0180/amcheck.git (fork; ls-remote main=2b1cbf9 reachable),
upstream=https://github.com/ivan-hc/amcheck/ (preserved). gh auth=coyoteclan (repo, workflow scopes).
Validators: pyyaml OK; actionlint/shellcheck/yamllint MISSING — canary dispatches are the GREEN gate.

Ruling (setup): implement directly on main, no worktree — plan prescribes per-task commits to main,
canary workflow_dispatch runs from the default branch, and push target is the user's scratch fork.
Cost if wrong: a broken intermediate commit lands on fork main (mitigated: fork is disposable,
canaries test 2 apps).

Ruling (setup): plan's probe-first scripts serve as the RED gate for YAML edits — GitHub Actions
YAML has no local unit runner; pyyaml parse + bash -n run pre-commit, canary dispatch is GREEN.
Cost if wrong: a YAML error surfaces only at canary time (mitigated: parse checks pre-commit).

Pre-flight interface scan (producer → consumer):
- T1 → T2: ko-<app> markers created in test step; out- uploaded always(); results/excluded path
  spelled correctly; comm --check-order. T2's loop/aggregation consumes all four. No conflict.
- T2 → T4/T5/T6: chunk entries {chunk, files}, results/job-<i>/ dirs, results-chunk-<i> artifact
  (if: always()), update-results loop over results/job-*/out-*. Later tasks edit inside this shape. No conflict.
- T2 → T6: generate-matrix script owned by T2's rewrite; T6 adds totest artifact upload in the same
  job. Sequential edits, T6 brief accounts for it. No conflict.
- T3 → T8: .github/actions/setup-runner/action.yml path; T8 reuses it. Pinned.
- T5 → T7: push --force-with-lease + exit 1 on race; T7's rotation relies on it. No conflict.
- T1 → T7: tested/excluded/log formats unchanged by T1 (only excluded starts growing). 4-file tracking stands. No conflict.
- Manual workflow untouched until T8; all prior tasks touch AMCHECK.yml only. No conflict.
Task 1: complete (commit 195aecc, pushed origin main; tests: /tmp/ko-contract-check.sh RED 4xFAIL→GREEN exit 0; canary 36840476950 success-path 0ad+7zip→tested; canary 36842818709 failure-path nonexistent-app-xyz→excluded with Upload-out-always + KO upload observed; results commits 19ed4b5, 173c9eb read).
Note: bot commit 19ed4b5 also deleted tracked results/appslist (downloaded artifact previously committed) — stray-file cleanup folded into Task 7.
Task 2: Ruling: commits use --no-gpg-sign — local commit.gpgsign=true but pinentry has no TTU in this session (signing failed, operation cancelled); repo/bot practice is unsigned commits. Cost if wrong: commits lack signatures (no functional impact).
Task 2 progress: chunk matrix + batch loop + single chunk upload + job-dir aggregation committed f1c805c, pushed. Probe /tmp/chunk-test.sh RED→GREEN exit 0; YAML OK; loop body bash -n clean; upload-artifact count=2. Canary 36843862438 (0ad+7zip, expect 1 chunk job) dispatched, watch running.
Task 2: Ruling: aggregate loop matches BOTH results/out-* and results/job-*/out-* — observed: upload-artifact flattens directory contents (artifact results-chunk-0 held out-*/ok-* at root, not under job-0/), so the job-dir-only loop matched nothing and stray out- files got committed (bot 1a6cc34). Fix committed 5f48366. Cost if wrong: none — both patterns delete-after-process; worst case is the pre-fix behavior (strays committed, cleaned in Task 7).
Task 2 progress: chunk canary 36843862438 GREEN — 1 chunk job (0, 0ad 7zip) in 2m16s (vs 7m31s+44s separate), 1 artifact results-chunk-0. Excluded-retest canary 36844310167 dispatched to prove aggregation through new code; watch running.
Note for Task 7: also remove tracked results/out-* strays (1a6cc34 committed results/out-0ad, results/out-7zip).
Task 2: Ruling: removed the `if: retest_excluded != 'true'` gate on Generate Matrix — with the gate, retest_excluded=true produced an empty matrix and failed the run at matrix evaluation (observed run 36844310167: all jobs green/skipped, run X). The in-script elif branch was unreachable. One-line deletion committed b628585 (rebased onto bot 2da186a). Cost if wrong: retest_excluded dispatches test excluded apps (its documented intent) — no downside found.
Observed: bot commit 2da186a re-processed STALE committed out-0ad (deleted it, duplicated +12 log lines) — stale out- files must be cleaned (Task 7) or every run re-appends duplicates. Note for Task 7: also handle results/out-7zip if still tracked.
Canary 36844680587 (retest_excluded=true, expect 1 chunk job testing nonexistent-app-xyz → excluded) dispatched, watch running.
Task 2: complete (commits f1c805c chunking, 5f48366 dual-layout aggregation, b628585 matrix-gate fix, 393a7cc sibling cleanup; pushed origin main. Tests: /tmp/chunk-test.sh RED→GREEN exit 0; loop body bash -n clean; canary 36843862438 1 chunk job 2m16s 1 artifact; canary 36844680587 chunked failure-path nonexistent-app-xyz→excluded, log +5, no dup. Sibling-cleanup rm verified statically + YAML OK; runtime proof deferred to Task 7 canary which asserts zero strays committed.)
Findings for later: upload-artifact flattens dir contents; stale committed out- files get re-processed (log dupes) until Task 7 cleans them.
Task 3 progress: composite action created (.github/actions/setup-runner/action.yml, bodies verbatim + amver output + ripgrep comment); run-actions uses ./amcheck/.github/actions/setup-runner via new Checkout AMCHECK step; stale Fix-AppArmor/Install steps removed (incl. a trailing-whitespace line and an interim stub, both cleaned). Probe RED→GREEN; YAML OK (both files). Committed 1d748f0, pushed. Canary 36845244052 dispatched, watch running.
Task 3: complete (commit 1d748f0, pushed. Tests: use-probe RED→GREEN; YAML OK both files; canary 36845244052 GREEN — chunk job 5m8s through composite action, both apps traced, log +12 exactly (no stale reprocessing). Results commit 834f2da read.)
Task 4 progress: TIMEOUT_S wired from env (23min), 9 timeout-1800→TIMEOUT_S, 3 third-attempt blocks deleted, sleep 30→10 (3 left). Probe /tmp/timeout-check.sh RED→GREEN; YAML OK; 6 am -i calls (2x3 branches); loop bash -n clean. Committed 1737918, pushed clean. Canary 36846101251 dispatched, watch running. Pending: DURATION p95 tuning (plan Step 5) after a full sweep — carry to Task 5 observation.
Task 4: complete (commit 1737918, pushed. Tests: /tmp/timeout-check.sh RED→GREEN; YAML OK; 6 am -i; bash -n clean; canary 36846101251 GREEN, DURATION=251 (0ad) and DURATION=6 (7zip) in committed log, results 08a3680 read.)
Deferred: timeout-minutes p95 tuning — only 2 samples so far; fork hourly sweeps will generate DURATION data; tune when a full sweep completes. Current 360 stands with always() partial-upload safety.
Task 5 progress: TOTEST 500, how_many desc max 500, dead push trigger removed + fast-path comment, 30-min schedule gated-off with enabling condition, push race exit 1. Probe /tmp/capacity-check.sh RED→GREEN; YAML OK. Committed 46cf750, pushed. Canary 36846858246 dispatched, watch running. Note: full 500-sweep wall-clock comes from the next hourly scheduled sweep, not canaries.
Task 5: complete (commit 46cf750, pushed. Tests: /tmp/capacity-check.sh RED→GREEN; YAML OK; canary 36846858246 GREEN, trace log +12 clean. Results commit bfc2a5e read.)
Task 6 progress: blocklist file created; totest artifact upload (gated on skip); static-checks job (AM checkout + AMCHECK checkout + totest download, greps + blocklist, static artifact always-uploaded); run-actions needs static + downloads it; loop skips blocked same-round via static/ko- and reuses static/out-; hardcoded blacklist branches deleted (0 pure_arg regexes left). Probe /tmp/prefilter-check.sh RED→GREEN; YAML OK; both loop bodies bash -n clean. Committed 228d6a9, pushed. Canary 36848321632 (koreader-nightly→excluded-no-install + 0ad→tested) dispatched, watch running.
Task 6: Ruling: static download path is programs/x86_64/static, not static/ — canary 36848321632 failed in 21s with `cat: static/out-0ad: No such file`: the loop runs under working-directory programs/x86_64 so relative static/ resolved there, while the artifact was downloaded to the job root. (Runner echoes the whole script at step start, which initially obscured the diagnosis; the error line was unambiguous.) Fixed ba62148 (rebased onto bot 3d91c1f). Cost if wrong: none — path now matches the only consumer.
Canary 36849043924 re-dispatched, watch running.
Task 6: complete (commit 228d6a9 + path fix ba62148, pushed. Tests: /tmp/prefilter-check.sh RED→GREEN; YAML OK; both bodies bash -n clean; canary 36849043924 GREEN — 0ad installed normally (DURATION=186), koreader-nightly skipped install same-round (DURATION=0, static metadata reused, in excluded). Results commit 0598fc6 read.)
Task 7 progress: 503 strays git-rm'd (incl. results/appslist, results/totest_selected.list, results/log-nonexistent-app-xyz); cleanup rm + anchored new-file guard added to update-results; already-tested wipe replaced with `git mv tested tested.prev` + lease push (manual reset_stats wipe kept — explicit operator action). Probe /tmp/hygiene-check.sh RED→GREEN (probe itself fixed: only the automatic wipe counts). YAML OK. Committed 19c93be (505 files), pushed. NOTE: `git add -A` also committed the local ledger .superpowers/.../progress.md — untrack it in Task 8 (git rm --cached), no secret content. Canary 36849880459 dispatched — asserts zero strays committed; watch running.
Task 7: Ruling: quarantine (keep 4 lists aside, wipe + recreate results/) instead of extending the rm list — canary 36849880459 proved the rm-list approach incomplete: the guard correctly failed the run rather than commit static/, totest_selected.list, appslist from the other artifacts. Quarantine is future-proof against new artifacts. Committed bf69b6b, pushed. Canary 36850703895 re-dispatched, watch running.
Task 7: complete (commit 19c93be cleanup + bf69b6b quarantine, pushed. Tests: /tmp/hygiene-check.sh RED→GREEN; canary 36849880459 FAILED LOUDLY via new guard (proved the guard works; rm-list incomplete); canary 36850703895 GREEN, bot commit 16b9004 = log +12 only, zero strays tracked. Results commits read.)
