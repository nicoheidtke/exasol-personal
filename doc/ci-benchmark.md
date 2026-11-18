# Comparing Go setup actions

Only the former “Push to PR or main” CI pipeline is compared. All four variants
are manual and use the same workload and job dependencies.

| Workflow file | Candidate | Pinned release |
| --- | --- | --- |
| `ci.yml` | `actions/setup-go` | v7.1.0 |
| `ci-ready.yml` | `nicoheidtke/ready-setup-go` | v2.0.0 |
| `ci-willabides.yml` | `willabides/setup-go-faster` plus `actions/cache` | v1.19.0 |
| `ci-cloudx.yml` | `cloudx-io/setup-go` | v1.0.0 |

The setup actions pin release commits. CloudX retains the installer/cache versions
bundled by its release. Use the same exact `go-version` input for every candidate
(default: 1.27.4). `GOTOOLCHAIN=local` prevents Go from silently switching compiler.
Linux uses `ubuntu-24.04`; Windows uses `windows-2025`. Runner image versions are
recorded in job summaries because image updates can still affect comparisons.

## Modes

With `full` unchecked (the default):

- Go and Terraform linting run normally.
- Linux and Windows compile the coverage-instrumented launcher.
- The Linux unit-test job generates sources and compiles all launcher tests with
  race detection and coverage, plus cleanup-module tests with their normal flags.
  `go test -c -o /dev/null` discards linked test binaries without executing test
  code, package initialization, or `TestMain`.
- Python linting is replaced with a placeholder. Python installation,
  documentation checks, and launcher test jobs are skipped.

Check `full` to execute the existing Go unit tests, Python lint/documentation
checks, CLI-reference check, and Linux/Windows launcher tests. There are no cloud
deployment tests or SonarQube scans in these comparison workflows. The launcher
test jobs consume the built executable; Go is installed only for coverage
conversion, with caching disabled. CloudX has no cache-off input in v1.0.0, so
this installation uses the same official setup-go commit CloudX uses internally.

Compile mode populates dependency and compilation caches, but does not populate
Go's successful test-result cache. Compare like modes. Use `full` for correctness
checks; its total duration includes test execution.

## Candidate configuration

Each candidate retains its native cache paths and can use its own sharing policy.
All dependency files are included, and OS/architecture partitioning is retained.
There is no blanket tool/module warmup in each job: each job builds the work it
needs. Cache cleanup between candidates prevents cross-candidate reuse without
forcing an identical cache layout on actions with different designs.

- Official setup-go uses its built-in combined caching on the normal Go paths.
  Lint hashes all module sums, builds hash the launcher sum, and unit tests hash
  launcher/cleanup sums. These native dependency inputs give each workload an
  appropriate snapshot instead of making coverage/race jobs consume only the
  first lint snapshot. Exact hits retain the existing snapshot.
- Ready uses restore/save, split module/build archives, zstd, and OS/architecture
  scopes. A shared build/test module scope lets trusted jobs reuse dependencies;
  compiled results keep the action's native per-job isolation. Default transfer
  concurrency is retained. Version 2 defaults to saving missing dependency
  baselines on the default branch; dependency-matching restores skip save-time
  scanning and uploads. Source-only changes do not refresh compiled results.
  Lint uses a separate module scope so its developer-tool dependencies do not
  inflate build/test transfers. Builds identify launcher dependencies; unit jobs
  identify launcher and cleanup dependencies, allowing a compatible shared module
  restore to seed a complete unit baseline instead of suppressing its save.
- WillAbides installs Go and exposes its cache paths to actions/cache. Module
  snapshots use workload-specific dependency keys and a shared build/test
  fallback, with a separate lint module group. Unit compilation can restore the
  launcher snapshot and save its additional dependencies under its own key;
  an exact launcher hit cannot suppress that save. Build snapshots
  are per job and source revision, with fallback to the previous revision.
  Unchanged-revision hits avoid uploads; a new revision can retain newly compiled
  results without repacking the module cache.
- CloudX uses a job-specific prefix and its default two-hour build-cache trimming.

Only the lint job caches linter analysis and Terraform plugins. These auxiliary
caches are also separated by candidate. The Python cache exists only in full runs.
Terraform plugin installation uses the built-in GitHub token to avoid sharing
the unauthenticated API quota of the runner's public IP address.
These configurations use each action's available features; cache sharing, trimming
and save frequency deliberately differ. Measure the complete cost and useful
reuse before concluding that a particular setting is faster.

## Manual cold and warm runs

In the fork's Actions tab, choose a “CI benchmark” workflow, click **Run workflow**,
select `main`, and leave `full` unchecked unless checking correctness.

Run one candidate batch at a time:

1. Wait for all runs to finish. Delete the fork's caches before the cold run,
   including caches on `main` that could otherwise provide fallback.
2. Run the candidate once and confirm cache misses and successful post-job saves.
3. Run it three times with unchanged inputs/source and confirm restores.
4. Record results, then clear caches before the next candidate.

For example, from a shell authenticated to this fork:

```sh
gh cache list --repo nicoheidtke/exasol-personal
# This deletes ALL caches in this fork, including Python and linter caches.
gh cache delete --all --repo nicoheidtke/exasol-personal
gh workflow run ci-ready.yml --repo nicoheidtke/exasol-personal --ref main -f full=false -f go-version=1.27.4
```

Cache deletion is a separate manual operation. The workflows never delete caches.
All four workflows share a concurrency group so only one benchmark runs at a time.
GitHub normally retains only one pending run in such a group: dispatch the next
run after the previous one completes, rather than submitting a whole batch.

A cold pipeline can have warm later jobs because caches produced by earlier jobs
may be eligible for reuse. Preserve that benefit where the action supports it;
confirm the actual restore logs. Deleting repository caches does not remove
preinstalled Go versions or image-provided files from hosted runners.

Repeat batches in a different candidate order. If storage grows near the fork's
configured cache limit, record that and check for unexpected misses/eviction;
even a single candidate can retain several snapshots. Keep runs on one source
revision and do not dispatch the other retained workflows during a comparison.

## Reading results

Compare setup/restore duration, post-job save duration, and their sum per Go job.
Include the runner's action preparation/download time when assessing total setup
overhead. Also record complete job time, pipeline elapsed time, and cache storage.
Compilation is enabled in both modes, so complete job/pipeline time includes it.
Exclude waiting for a runner when comparing job execution, and report queue delays
separately if using dispatch-to-completion elapsed time.

Check successful restores and saves in logs instead of comparing `cache-hit`
booleans across actions: their meanings differ, and snapshot-based caches normally
restore using prefixes. Cache errors and unexpected misses invalidate a warm-run
measurement. Use medians of repeated runs; a source edit and a dependency edit are
useful follow-up batches after the unchanged-source comparison.
