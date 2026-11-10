# alcf-ci

Run the tests of a GitHub repository on ALCF machines (Aurora) and get the result back on
the commit and the pull request, without a self-hosted GitHub runner and without a hosted
runner waiting for the batch job.

The repository is tested by [ALCF GitLab CI](https://docs.alcf.anl.gov/services/gitlab-ci/)
on a mirror of the GitHub repository. This repository holds the shared parts of that
bridge, so that a project only carries three short files:

| Shared here | Purpose | In the project |
|---|---|---|
| [`.github/workflows/trigger.yml`](.github/workflows/trigger.yml) | reusable workflow: start the GitLab pipeline, set the commit status to pending | `.github/workflows/alcf.yml` ([example](examples/github/alcf.yml)) |
| [`.github/workflows/report.yml`](.github/workflows/report.yml) | reusable workflow: set the final commit status, copy the job logs | `.github/workflows/alcf-report.yml` ([example](examples/github/alcf-report.yml)) |
| [`gitlab/aurora-julia.yml`](gitlab/aurora-julia.yml) | GitLab CI template: Julia on Aurora, build on the login node, test in PBS | `.gitlab-ci.yml` ([example](examples/gitlab/.gitlab-ci.yml)) |

The two workflows work for any GitLab CI pipeline on the ALCF instance. The GitLab template
is for Julia packages on Aurora; a project with other needs writes its own `.gitlab-ci.yml`
and still uses the workflows.

The setup was extracted from the copies in
[JuliaGPU/oneAPI.jl](https://github.com/JuliaGPU/oneAPI.jl) and
[exanauts/SparseDirectSolver.jl](https://github.com/exanauts/SparseDirectSolver.jl).

## Contents

- [How it works](#how-it-works)
- [Requirements](#requirements)
- [Setting up a project](#setting-up-a-project)
- [Reference: trigger workflow](#reference-trigger-workflow)
- [Reference: report workflow](#reference-report-workflow)
- [Reference: GitLab template](#reference-gitlab-template)
- [Day-to-day use](#day-to-day-use)
- [Maintenance](#maintenance)
- [Limits](#limits)
- [Trust](#trust)
- [Versioning](#versioning)
- [Migrating a project that carries its own copy](#migrating-a-project-that-carries-its-own-copy)

## How it works

```
push / pull request / label on a fork pull request
        │
        ▼
alcf.yml → trigger.yml (about a minute)      GitLab pipeline on Aurora (minutes to hours)
  mirror sync, or push of the fork head ────►  build (login node) → test (PBS batch job)
  commit status: pending                                 │ every status change
                                                         ▼
                                             project webhook (pipeline events)
                                                         │ workflow_dispatch
                                                         ▼
                                             alcf-report.yml → report.yml (about a minute,
                                             final states only)
                                               read the pipeline back from GitLab
                                               commit status: success / failure / error
                                               copy the job logs into the run
```

The result is a **commit status** (default name `ALCF / Aurora`) on the tested commit. It
shows up in the checks of a pull request and can be made a required check.

Properties of the design:

- **No runner waits.** Both workflows finish in about a minute. The batch job can sit in the
  PBS queue for hours without holding a GitHub runner or hitting a job timeout.
- **Fork code is opt-in.** GitLab CI jobs run as an ALCF user on the machine. Pushes and
  pull requests from branches of the repository run automatically; a pull request from a
  fork runs only after a maintainer applies a label (default `alcf-ci`), and again only
  when the label is re-applied after new commits. The trigger workflow only moves git refs;
  it never checks out or executes pull-request code.
- **The webhook cannot forge a result.** The dispatch inputs are only a hint; the report
  workflow reads the commit and the status back from GitLab.
- **Only the newest pipeline of a commit speaks for it**, so a late report of an old
  pipeline does not replace the result of a re-run.
- **Infrastructure failures are not test failures.** A pipeline whose failed jobs never ran
  their script (no runner online, a batch job PBS never started, a runner error) is retried
  once with a fresh pipeline, and reported as `error` naming the infrastructure if the retry
  fails the same way.
- **Logs are readable without an ALCF login.** The GitLab instance needs an ALCF account,
  so the report run contains the job logs and the commit status links to that run.

Two things about ALCF's Jacamar runners shape the token setup: they refuse pipelines
attributed to a bot user ("bot account token used for job, unsupported by runner"), and a
project access token *is* a bot user. Pipelines therefore start in one of two ways only: by
the mirror sync (attributed to the user who configured the mirror) or through a pipeline
trigger token (attributed to the user who created it). The project access token is used for
API calls and for moving refs, never to create a pipeline.

## Requirements

- An ALCF account with an allocation on the machine, access to
  [ALCF GitLab CI](https://docs.alcf.anl.gov/services/gitlab-ci/), and the right to create
  a project there. CI jobs run as the user who owns the mirror and the trigger token.
- A public GitHub repository (the mirror and the fork push fetch from it without
  credentials) in which you can add workflows, secrets and labels.
- The GitHub organization must allow reusable workflows from `anlsys/alcf-ci` (Settings →
  Actions → General → "Allow actions and reusable workflows"; nothing to do if all actions
  are allowed).
- For the GitLab template: a shared filesystem path for the Julia depot and a site `julia`
  module of the wanted version.

## Setting up a project

Placeholders: `ORG/REPO` is the GitHub repository, `USER/PROJECT` the GitLab mirror project
on `gitlab-ci.alcf.anl.gov`.

### 1. GitLab: the mirror project

1. Create the project `USER/PROJECT` (private is fine).
2. Settings → Repository → Mirroring repositories: add `https://github.com/ORG/REPO.git`,
   direction **Pull**, with
   - **Overwrite diverged branches** on (force-pushed branches follow),
   - **Trigger pipelines for mirror updates** on (this is what starts a pipeline),
   - **Only mirror protected branches** off (pull-request branches must be mirrored).

   Do this as the ALCF user the jobs should run as: mirror pipelines are attributed to the
   user who set up the mirror.
3. If the jobs' git fetch fails on the instance's certificate, add the CI/CD variable
   `GIT_SSL_NO_VERIFY` = `true` (Settings → CI/CD → Variables). The existing mirror
   projects set it.

### 2. GitLab: two tokens

1. **Project access token** (Settings → Access tokens): role **Maintainer**, scopes **api**
   and **write_repository**. Used to start a mirror sync, read pipelines, jobs and logs,
   and push fork heads as `gh-pr-<N>` branches. Note the expiry date.
2. **Pipeline trigger token** (Settings → CI/CD → Pipeline trigger tokens), created by the
   ALCF user the jobs should run as. Used for fork pull requests, re-runs after a failed
   pipeline and the automatic retry of infrastructure failures.

### 3. GitHub: secrets and label

In `ORG/REPO` (Settings → Secrets and variables → Actions), or as organization secrets:

| Secret | Value |
|---|---|
| `ALCF_GITLAB_TOKEN` | the project access token |
| `ALCF_GITLAB_TRIGGER_TOKEN` | the pipeline trigger token |

Create the label `alcf-ci`:

```sh
gh label create alcf-ci -R ORG/REPO --description "Run this fork pull request on ALCF"
```

### 4. The three files

Copy and fill in the placeholders:

| From | To |
|---|---|
| [`examples/github/alcf.yml`](examples/github/alcf.yml) | `.github/workflows/alcf.yml` |
| [`examples/github/alcf-report.yml`](examples/github/alcf-report.yml) | `.github/workflows/alcf-report.yml` |
| [`examples/gitlab/.gitlab-ci.yml`](examples/gitlab/.gitlab-ci.yml) | `.gitlab-ci.yml` |

`gitlab_project` and `context` must be the same in both workflows. `alcf-report.yml` has to
be on the default branch before the webhook can call it, so merge these files first; the
first run of `alcf.yml` then leaves a pending status until step 5 is done.

### 5. GitLab: the webhook

1. Create a GitHub **fine-grained personal access token**: resource owner `ORG`, repository
   `ORG/REPO` only, permission **Actions: read and write**, nothing else. This permission
   can start workflows but cannot push code. Note the expiry date.
2. In the GitLab project, Settings → Webhooks → Add new webhook:
   - URL: `https://api.github.com/repos/ORG/REPO/actions/workflows/alcf-report.yml/dispatches`
   - Trigger: **Pipeline events** only; SSL verification enabled
   - Custom headers: `Authorization` = `Bearer <the token>` and
     `Accept` = `application/vnd.github+json`
   - Custom webhook template (`main` is the default branch of `ORG/REPO`):

     ```json
     {"ref":"main","inputs":{"pipeline_id":"{{object_attributes.id}}","status":"{{object_attributes.status}}"}}
     ```

   The token lives in the webhook configuration only, so CI jobs, including those of
   label-gated fork pull requests, never see it.
3. Test → Pipeline events. "Recent events" must show a 2xx response and a run must appear
   under *ALCF CI report* in the Actions tab of `ORG/REPO`.

### 6. Check it

- Report a finished pipeline by hand; the commit it tested gets the status:

  ```sh
  gh workflow run alcf-report.yml -R ORG/REPO -f pipeline_id=<id>
  gh api repos/ORG/REPO/commits/<sha>/status --jq '.statuses[] | {context, state, description}'
  ```

- Open a pull request from a branch of the repository: `ALCF CI` finishes within a few
  minutes, the status is pending and links to the pipeline, and the final status arrives
  when the pipeline finishes.
- Once the status has been reported, it can be added as a required status check to the
  branch ruleset under the name given as `context`.

## Reference: trigger workflow

`anlsys/alcf-ci/.github/workflows/trigger.yml`, called from a workflow that triggers on
`push` (the branches to test), `pull_request` and `pull_request_target` with
`types: [labeled]`.

| Input | Default | |
|---|---|---|
| `gitlab_project` | required | path of the mirror project, `USER/PROJECT` |
| `gitlab_host` | `gitlab-ci.alcf.anl.gov` | GitLab instance |
| `context` | `ALCF / Aurora` | name of the commit status |
| `label` | `alcf-ci` | label that runs a fork pull request |
| `report_workflow` | `alcf-report.yml` | file name of the caller's report workflow |

| Secret | |
|---|---|
| `gitlab_token` | required; the project access token |
| `trigger_token` | the pipeline trigger token; without it fork pull requests and re-runs after a failed pipeline fail with a message |

Caller permissions: `statuses: write`, `actions: write`.

What it does, by event:

| Event | Action |
|---|---|
| `push`, `pull_request` from a branch of the repository | asks GitLab to sync the mirror now, which creates the pipeline |
| `pull_request` from a fork | nothing (the job is skipped) |
| `pull_request_target`, the label applied to a fork pull request | checks that the head is still the labeled commit, pushes it to GitLab as `gh-pr-<N>` with `ci.skip`, creates the pipeline with the trigger token |
| re-run of a run whose newest pipeline failed, was canceled or skipped | creates a fresh pipeline with the trigger token, if the commit is still the tip of its branch on GitLab |

It then waits up to 10 minutes for the pipeline to exist, sets the pending status with a
link to it (or to the run, if the mirror has not synced yet), and dispatches the report
workflow itself if the pipeline had already finished. A failed mirror-sync request is only
a warning: the scheduled sync still creates the pipeline, later.

Runs for the same pull request or branch cancel each other, so only the newest commit is
handled.

## Reference: report workflow

`anlsys/alcf-ci/.github/workflows/report.yml`, called from a workflow that triggers on
`workflow_dispatch` with the inputs `pipeline_id` and `status`.

| Input | Default | |
|---|---|---|
| `pipeline_id` | required | GitLab pipeline id |
| `status` | empty | status sent by the webhook; a run with an empty status always reports |
| `gitlab_project` | required | path of the mirror project |
| `gitlab_host` | `gitlab-ci.alcf.anl.gov` | GitLab instance |
| `context` | `ALCF / Aurora` | name of the commit status |
| `label` | `alcf-ci` | named in the failure summary |
| `publish_logs` | `true` | copy the GitLab job logs into the run |

| Secret | |
|---|---|
| `gitlab_token` | required; the project access token |
| `trigger_token` | the pipeline trigger token; without it an infrastructure failure is reported, not retried |

Caller permissions: `statuses: write`.

How a pipeline maps to the commit status:

| Pipeline | Status | |
|---|---|---|
| `success` | `success` | |
| `failed`, a job ran and failed | `failure` | the description names the failed jobs |
| `failed`, every failed job has an infrastructure reason | `pending`, then `error` | retried once as a fresh pipeline; `error` if that fails the same way |
| `canceled` | `error` | |
| `skipped` (`[skip ci]`) | unchanged | |
| still running (manual run) | `pending` | links to the pipeline |
| not the newest pipeline of its commit | unchanged | |

The infrastructure reasons are `stuck_pending_no_matching_runners`, `no_updates_running`,
`runner_system_failure`, `stuck_or_timeout_failure`, `scheduler_failure` and
`runner_unsupported`. Jobs with `allow_failure` do not count.

The webhook fires on every status change of a pipeline. Runs for non-final states skip
their job and use no runner, but appear as skipped runs in the Actions tab. The run itself
is attached to the default branch and is green whenever reporting worked, also for a failed
pipeline; the pipeline's result is in the commit status.

## Reference: GitLab template

[`gitlab/aurora-julia.yml`](gitlab/aurora-julia.yml), included with

```yaml
include:
  - remote: 'https://raw.githubusercontent.com/anlsys/alcf-ci/v1/gitlab/aurora-julia.yml'
```

It includes ALCF's runner definitions (`anl/ci-resources/defaults`), so the project does
not have to.

Variables the project sets globally (top-level `variables:`):

| Variable | |
|---|---|
| `JULIA_VERSION` | version of the site `julia` module, e.g. `"1.13"` |
| `JULIA_DEPOT_PATH` | persistent depot on a filesystem the compute nodes mount; one per project and Julia version |

Variables the template sets:

| Variable | Value | |
|---|---|---|
| `JULIA_CPU_TARGET` | `generic;icelake-server;sapphirerapids,-avx512fp16` | package images for login and compute nodes; must be the same in every job |
| `JULIA_JIT_TARGET` | `native,-avx512fp16` | for `julia -C "$JULIA_JIT_TARGET"`; `JULIA_CPU_TARGET` does not constrain the runtime JIT |

Job templates:

| Template | Runs | Use |
|---|---|---|
| `.aurora-julia-build` | login node (shell runner, has network), stage `build`, timeout 2 h | `Pkg.instantiate`, `Pkg.precompile`, anything that downloads; pass manifests on as artifacts |
| `.aurora-julia-test` | PBS batch job (no network), stage `test`, timeout 5 h including the queue wait | the tests; set `ANL_AURORA_SCHEDULER_PARAMETERS` |
| `.aurora-env` | script fragment, already the `before_script` of both templates | loads `julia/$JULIA_VERSION`, fails early if the two variables are missing |
| `.aurora-drop-site-oneapi` | script fragment | removes the site oneAPI libraries from `LD_LIBRARY_PATH`; for packages whose SYCL runtime comes from Julia artifacts or Conda |

Also set: `workflow: rules` (pipelines for push, web, api and trigger sources),
`default: interruptible: true` and `stages: [build, test]`. Both job templates use the
resource group `aurora-ci`, which serializes the jobs of a project across its pipelines.

A job that overrides `before_script` must include the environment itself:

```yaml
  before_script:
    - !reference [.aurora-env, script]
    - export MY_SETTING=1
```

Several Julia versions: define one hidden job per version that sets `JULIA_VERSION` and
`JULIA_DEPOT_PATH`, and spell the jobs out (`extends: [.aurora-julia-test, .julia-1.13]`,
`needs: ["build:1.13"]`). `parallel: matrix` does not work here, because GitLab does not
expand variables in `needs:parallel:matrix`.

## Day-to-day use

| Want | Do |
|---|---|
| Run a pull request from a fork | apply the `alcf-ci` label after reading the diff; re-apply it after new commits |
| Run a commit again | re-run the `ALCF CI` run of that commit; a fresh pipeline is created if the newest one did not succeed |
| A status is stuck on pending | `gh workflow run alcf-report.yml -f pipeline_id=<id>` (the id is in the status link or on GitLab) |
| Skip a commit | `[skip ci]` in the commit message skips both sides |
| Read the logs | the commit status links to the report run, which contains the GitLab job logs |

## Maintenance

Three credentials expire, each with a different symptom:

| Credential | Where | Symptom when expired |
|---|---|---|
| GitHub fine-grained token | GitLab webhook header | statuses stay pending; GitLab disables the webhook after repeated failed deliveries |
| GitLab project access token | secret `ALCF_GITLAB_TOKEN` | the trigger run warns that there is no pipeline yet, the report run fails with "could not read GitLab pipeline" |
| GitLab trigger token | secret `ALCF_GITLAB_TRIGGER_TOKEN` | fork pull requests and re-runs fail; trigger tokens do not expire by default, but die with their owner's access |

After renewing the webhook token, re-enable the webhook and report the missed pipelines by
hand. GitLab does not repeat a failed delivery.

Expiry dates of the GitLab side:

```sh
glab api projects/USER%2FPROJECT/access_tokens | jq '.[] | {name, expires_at, active}'
glab api projects/USER%2FPROJECT/hooks | jq '.[] | {url, alert_status}'
```

## Limits

- **One queued job per user and queue.** The Aurora debug queues admit a single queued job
  per user; a second `qsub` is refused, which shows up as a failed job. The resource group
  serializes the jobs of one project, but not of two projects of the same ALCF user. Give
  such projects different queues (`debug`, `debug-scaling`), or expect occasional refused
  submissions when both have pipelines at the same time.
- **1 h walltime** in the debug queues. Keep downloads and precompilation in the build job.
- **Public GitHub repositories only.** The mirror and the fork push fetch without
  credentials.
- **Job logs become as visible as the repository's Actions runs.** Set `publish_logs:
  false` if the logs may contain something that should stay behind the ALCF login.
- **The check appears twice in a pull request**: the short `ALCF CI` workflow run (did the
  pipeline start) and the commit status (the result). Only the commit status says whether
  the tests passed.
- **Pull requests from forks are not updated automatically**; each new head needs the label
  again.

## Trust

Both sides run code from this repository with the project's credentials:

- the reusable workflows receive the GitLab tokens of the calling project, and
- the GitLab template is executed as the ALCF user of the mirror project.

Whoever can push to the ref a project uses can therefore act as that project on ALCF. A
project that does not want to follow `v1` pins both references to a commit:

```yaml
    uses: anlsys/alcf-ci/.github/workflows/trigger.yml@<commit sha>
```

```yaml
  - remote: 'https://raw.githubusercontent.com/anlsys/alcf-ci/<commit sha>/gitlab/aurora-julia.yml'
```

## Versioning

`v1` is a moving tag: fixes and compatible additions are released by moving it, so projects
on `@v1` get them without a change of their own. A change that needs action in the projects
(a renamed input, a changed commit-status name, a different token setup) gets a new major
tag. `main` is where changes land first; test them from one project with `@main` before
moving the tag.

## Migrating a project that carries its own copy

For a repository that already has the full `alcf.yml` and `alcf-report.yml`:

1. Replace both files with the examples. Keep the file names, the `context` the project
   already uses (e.g. `ALCF / LTS stack (Aurora)`), and set `gitlab_project`. The numeric
   project id is no longer needed.
2. Nothing changes on GitLab: the mirror, the tokens, the secrets and the webhook stay as
   they are.
3. In `.gitlab-ci.yml`, replace the include of `anl/ci-resources/defaults`, the `workflow`,
   `default`, CPU-target variables and `.aurora-env` with the include of the template, and
   let the jobs extend `.aurora-julia-build` and `.aurora-julia-test`. Project-specific
   stages (e.g. coverage) are declared by overriding `stages`.
4. The job of the trigger run is now named `<caller job> / Start pipeline`. The commit
   status keeps its name, so a ruleset that requires the status needs no change; one that
   requires the old job name does.
