# Data ingest runner

Self-hosted runner for https://github.com/brain-bbqs/data-ingest-task-force operations.

Two workflows run on this runner, named after how they're triggered: **Scheduled Ingest** (`.github/workflows/scheduled_ingest.yml`, the daily cron) and **Manual Ingest** (`.github/workflows/manual_ingest.yml`, on-demand from the Actions tab, with `dry_run`/`only`/`task_force_ref` inputs). Both are thin trigger wrappers around the reusable `.github/workflows/ingest.yml`, which holds the actual job; they share a `concurrency` group, so a manual run and a scheduled run never ingest at the same time.

The reusable workflow checks out [`data-ingest-task-force`](https://github.com/brain-bbqs/data-ingest-task-force) and runs its `dispatch/dispatch.py` (downloads each lab's incoming dandiset, converts new sessions, uploads the standardized outputs).

## Runner setup

1. Register a self-hosted runner against this repository (Settings -> Actions -> Runners -> New self-hosted runner), giving it (in addition to the default `self-hosted` label) the labels the workflow targets: `ember` and `ingest` — or edit `runs-on` in `ingest.yml` (the one place it's declared) to match whatever labels you use.
2. On the runner host, install and configure:
   - Python 3.10+ (`python3` on `PATH`). `dispatch.py` is the orchestrator, not a payload: it runs on the host and starts a container per step, so the host needs an interpreter for it even though nothing it drives runs here. It's standard-library-only, so there's no environment to build for it — no `pip install`, no venv.
   - Docker. Every external tool `dispatch.py` drives runs in a container, not directly on the runner host: `dandi download`/`dandi upload` run inside `ghcr.io/brain-bbqs/dandi-cli` (dispatch's own `--dandi-image` default), and each lab's conversion step runs inside its own registered image (e.g. [`ghcr.io/brain-bbqs/kemere-r34da059514-ingest`](https://github.com/brain-bbqs/data-ingest-task-force/pkgs/container/kemere-r34da059514-ingest)) — code and data are bind-mounted in at run time, so the runner host doesn't need the `dandi` CLI, FFmpeg, or any other lab runtime dependency installed directly. Both images are currently public, so no `docker login` is needed yet; a future private image would need `docker login ghcr.io` run once on the runner.
   - Set the `EMBER_DANDI_API_KEY` **repository secret** (Settings -> Secrets and variables -> Actions -> Secrets) to a DANDI API key for the `ember-dandi` instance. The workflow exposes it to `dispatch.py` under that same name (no rename) — dandi-cli looks credentials up per instance (upper-cased, `-` → `_`, suffixed `_API_KEY`, so `ember-dandi` needs `EMBER_DANDI_API_KEY`, not a generic `DANDI_API_KEY`), and `dispatch.py` computes that name itself and forwards it (by name only, never as a literal value) into every container it starts — the dandi image and, when set, a project's `container_image` too, for labs whose conversion step itself needs DANDI access. Since `dandi` only ever runs inside its container now, there's no host-side `dandi login` fallback — this secret is the only way to authenticate. A lab registered against a different `dandi_instance` needs its own correspondingly-named secret.
   - Set the `MAIL_USERNAME` and `MAIL_PASSWORD` **repository secrets** (a Gmail address and an [app password](https://support.google.com/accounts/answer/185833) for it) — **Scheduled Ingest** emails through `smtp.gmail.com` when a run fails or never starts because this runner is offline. Only that notification job needs them; ingest itself runs without.
3. Nothing else required here: `dispatch.py` always picks and creates the raw/standardized folders itself (`ember-incoming`/`ember-standardized`, siblings of its own checkout on the runner) — no workflow here has a variable or input that overrides that location.

By default dispatch reads its lab registry (`projects.json` + `sessions.json`) from the task-force checkout, so adding or editing a lab needs a commit + PR there.
To edit the registry directly on the runner host instead — no PR needed — set two more repository variables to absolute paths on this runner:
   - `REGISTRY_PATH` (a `projects.json`)
   - `SESSIONS_PATH` (a `sessions.json`)

When set, dispatch reads those files instead of the repo's committed copies; leave them unset to keep the registry version-controlled in `data-ingest-task-force`.
See `dispatch/README.md` in that repo for the file formats (and their JSON Schemas, for editor validation).

## Running it

- **Scheduled Ingest** fires automatically on its `schedule:` cron (daily by default — adjust in `scheduled_ingest.yml` to match how often new sessions are expected). It takes no inputs: a scheduled run is always a full, non-dry run. If it fails, its `NotifyOnFailure` job emails a notification (on a GitHub-hosted runner, so the mail goes out even when the failure is this self-hosted runner) — manual runs don't email. The same job also catches this runner being offline: an offline runner never fails anything, its job just sits queued until GitHub cancels it 24 hours later, so `NotifyOnFailure` fires on a cancelled run too, with a "Runner Offline" subject instead of "CI Failure". Expect that mail once the first scheduled run has waited out its day (so one to two days into an outage), then daily until the runner is back.
- **Manual Ingest** is the one to trigger by hand from the Actions tab, optionally with `dry_run: true` (log every action, touch nothing) or `only: <lab>` to restrict to one project. Its run title notes the lab and dry-run state, so a manual run is distinguishable from a scheduled one at a glance.

By default every run checks out `data-ingest-task-force`'s `main` branch. For debugging, override that per-run with Manual Ingest's `task_force_ref` input (a branch, tag, or SHA), or set it as a standing override via the `TASK_FORCE_REF` repository variable — the input wins if both are set, and it falls back to `main` if neither is. Don't leave `TASK_FORCE_REF` pointed at a feature branch for real (non-debugging) scheduled runs; it's meant to come back out once you're done.

## Why doesn't this repository allow pull requests from external forks?

This repository uses a self-hosted runner instead of GitHub-hosted Actions runners.
This means that external users could fork and submit a pull request that contains code modifications that might expose secrets or other hostile actions.
While this could be mitigated through careful permissioning and approval of run triggers before accepting contributions, it is much safer overall to simply disable them.
If you have any questions or suggestions, please raise an Issue instead.
The repository is kept public to allow anyone to see the runtime logs of the submission process, as well as the success/failure/timestamp of the triggers.
