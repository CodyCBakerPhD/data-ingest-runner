# Data ingest runner

Self-hosted runner for https://github.com/brain-bbqs/data-ingest-task-force operations.

`.github/workflows/cron_ingest.yml` runs on this runner on a schedule (and on demand via `workflow_dispatch`).
It checks out [`data-ingest-task-force`](https://github.com/brain-bbqs/data-ingest-task-force) and runs its `dispatch/dispatch.py` — see that repo's `dispatch/README.md` for what a run actually does (download each lab's incoming dandiset, convert new sessions, upload the standardized output).

## Runner setup

1. Register a self-hosted runner against this repository (Settings -> Actions -> Runners -> New self-hosted runner), giving it (in addition to the default `self-hosted` label) the label the workflow targets: `data-ingest` — or edit `runs-on` in `cron_ingest.yml` to match whatever label you use.
2. Create local top-level folders for the raw and standardized dandiset copies (e.g. `/data/ember-incoming`, `/data/ember-standardized`), and set them as repository variables so the workflow can find them without the paths being hardcoded in the workflow file: Settings -> Secrets and variables -> Actions -> Variables ->
   - `EMBER_INCOMING_DIR`
   - `EMBER_STANDARDIZED_DIR`

## Running it

- Fires automatically on the workflow's `schedule:` cron (hourly by default — adjust in `cron_ingest.yml` to match how often new sessions are expected).
- Trigger on demand from the Actions tab (`workflow_dispatch`), optionally with `dry_run: true` (log every action, touch nothing) or `only: <lab>` to restrict to one project.

## Why doesn't this repository allow pull requests from external forks?

This repository uses a self-hosted runner instead of GitHub-hosted Actions runners.
This means that external users could fork and submit a pull request that contains code modifications that might expose secrets or other hostile actions.
While this could be mitigated through careful permissioning and approval of run triggers before accepting contributions, it is much safer overall to simply disable them.
If you have any questions or suggestions, please raise an Issue instead.
The repository is kept public to allow anyone to see the runtime logs of the submission process, as well as the success/failure/timestamp of the triggers.
