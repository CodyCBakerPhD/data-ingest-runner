# Data ingest runner

Self-hosted runner for https://github.com/brain-bbqs/data-ingest-task-force operations.

`.github/workflows/cron_ingest.yml` runs on this runner on a schedule (and on demand via `workflow_dispatch`).
It checks out [`data-ingest-task-force`](https://github.com/brain-bbqs/data-ingest-task-force) and runs its `dispatch/dispatch.py` — see that repo's `dispatch/README.md` for what a run actually does (download each lab's incoming dandiset, convert new sessions, upload the standardized output).

## Runner setup

1. Register a self-hosted runner against this repository (Settings -> Actions -> Runners -> New self-hosted runner), giving it (in addition to the default `self-hosted` label) the labels the workflow targets: `ember` and `ingest` — or edit `runs-on` in `cron_ingest.yml` to match whatever labels you use.
2. On the runner host, install and configure:
   - Python 3.10+ (`python3` on `PATH`). `dispatch.py` itself is standard-library-only, so nothing else needs installing for it to run.
   - The [`dandi` CLI](https://github.com/dandi/dandi-cli), logged in for every archive instance named in `dispatch/projects.json` (`dandi login -i emberarchive`, run once — the workflow does not manage credentials).
   - Docker. Each lab's conversion step runs inside its own registered image (e.g. [`ghcr.io/brain-bbqs/kemere-r34da059514-ingest`](https://github.com/brain-bbqs/data-ingest-task-force/pkgs/container/kemere-r34da059514-ingest), currently public — no `docker login` needed), which holds only that lab's runtime environment (FFmpeg, etc.) — code and data are bind-mounted in at run time, so the runner host itself doesn't need those dependencies installed directly. A future private image would need `docker login ghcr.io` run once on the runner.
3. Create local top-level folders for the raw and standardized dandiset copies (e.g. `/data/ember-incoming`, `/data/ember-standardized`), and set them as repository variables so the workflow can find them without the paths being hardcoded in the workflow file: Settings -> Secrets and variables -> Actions -> Variables ->
   - `EMBER_INCOMING_DIR`
   - `EMBER_STANDARDIZED_DIR`

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
