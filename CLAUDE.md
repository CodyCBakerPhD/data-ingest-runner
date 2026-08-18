# Agent instructions

## Commits and PRs

- Always link PRs to issues when possible
- PR titles should be human-readable and in the past tense. They should NOT use conventional commit style.
- Keep PR descriptions as short and concise as possible: the fewest words that describe the change accurately
- End every PR description with a `<details>` dropdown holding the prompts that asked for the work, quoted verbatim and in order: the original request first, then each follow-up as the branch grows. The prose above it stays a description of the change, not of the conversation
- Every commit must include a `Co-Authored-By` trailer identifying your tool name and version and your underlying model and version. Format (replace all `<…>` placeholders with actual values): `Co-Authored-By: <Tool> <tool-version> / <Model> <model-version> <noreply@vendor-domain>`
