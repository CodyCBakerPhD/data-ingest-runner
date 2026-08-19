# Data ingest runner

Self-hosted runner for https://github.com/brain-bbqs/data-ingest-task-force operations.

`.github/workflows/cron_ingest.yml` runs on this runner on a schedule (and on demand via `workflow_dispatch`).
It checks out [`data-ingest-task-force`](https://github.com/brain-bbqs/data-ingest-task-force) and runs its `dispatch/dispatch.py` — see that repo's `dispatch/README.md` for what a run actually does (download each lab's incoming dandiset, convert new sessions, upload the standardized output).

## Runner setup

1. Register a self-hosted runner against this repository (Settings -> Actions -> Runners -> New self-hosted runner), giving it (in addition to the default `self-hosted` label) the labels the workflow targets: `ember` and `ingest` — or edit `runs-on` in `cron_ingest.yml` to match whatever labels you use.
2. On the runner host, install and configure:
   - Python 3.10+ (`python3` on `PATH`). `dispatch.py` itself is standard-library-only, so nothing else needs installing for it to run.
   - Docker. Every external tool `dispatch.py` drives runs in a container, not directly on the runner host: `dandi download`/`dandi upload` run inside `ghcr.io/brain-bbqs/dandi-cli` (dispatch's own `--dandi-image` default), and each lab's conversion step runs inside its own registered image (e.g. [`ghcr.io/brain-bbqs/kemere-r34da059514-ingest`](https://github.com/brain-bbqs/data-ingest-task-force/pkgs/container/kemere-r34da059514-ingest)) — code and data are bind-mounted in at run time, so the runner host doesn't need the `dandi` CLI, FFmpeg, or any other lab runtime dependency installed directly. Both images are currently public, so no `docker login` is needed yet; a future private image would need `docker login ghcr.io` run once on the runner.
   - Set the `EMBER_DANDI_API_KEY` **repository secret** (Settings -> Secrets and variables -> Actions -> Secrets) to a DANDI API key for the `emberarchive` instance. The workflow exposes it to `dispatch.py` as `DANDI_API_KEY`, which forwards it (by name only, never as a literal value) into every container it starts — the dandi image and, when set, a project's `container_image` too, for labs whose conversion step itself needs DANDI access. Since `dandi` only ever runs inside its container now, there's no host-side `dandi login` fallback — this secret is the only way to authenticate. Verify against the `dandi` CLI version baked into the dandi image whether a plain `DANDI_API_KEY` env var authenticates a *named* instance (`-i emberarchive`) the way you expect.
3. Nothing else required here: `dispatch.py` always picks and creates the raw/standardized folders itself (`ember-incoming`/`ember-standardized`, siblings of its own checkout on the runner) — this workflow has no variable or input that overrides that location.

By default dispatch reads its lab registry (`projects.json` + `sessions.json`) from the task-force checkout, so adding or editing a lab needs a commit + PR there.
To edit the registry directly on the runner host instead — no PR needed — set two more repository variables to absolute paths on this runner:
   - `REGISTRY_PATH` (a `projects.json`)
   - `SESSIONS_PATH` (a `sessions.json`)

When set, dispatch reads those files instead of the repo's committed copies; leave them unset to keep the registry version-controlled in `data-ingest-task-force`.
See `dispatch/README.md` in that repo for the file formats (and their JSON Schemas, for editor validation).

## Running it

- Fires automatically on the workflow's `schedule:` cron (daily by default — adjust in `cron_ingest.yml` to match how often new sessions are expected).
- Trigger on demand from the Actions tab (`workflow_dispatch`), optionally with `dry_run: true` (log every action, touch nothing) or `only: <lab>` to restrict to one project.

By default every run checks out `data-ingest-task-force`'s `main` branch. For debugging, override that per-run with the `task_force_ref` `workflow_dispatch` input (a branch, tag, or SHA), or set it as a standing override via the `TASK_FORCE_REF` repository variable — the input wins if both are set, and it falls back to `main` if neither is. Don't leave `TASK_FORCE_REF` pointed at a feature branch for real (non-debugging) scheduled runs; it's meant to come back out once you're done.

## Why doesn't this repository allow pull requests from external forks?

This repository uses a self-hosted runner instead of GitHub-hosted Actions runners.
This means that external users could fork and submit a pull request that contains code modifications that might expose secrets or other hostile actions.
While this could be mitigated through careful permissioning and approval of run triggers before accepting contributions, it is much safer overall to simply disable them.
If you have any questions or suggestions, please raise an Issue instead.
The repository is kept public to allow anyone to see the runtime logs of the submission process, as well as the success/failure/timestamp of the triggers.
