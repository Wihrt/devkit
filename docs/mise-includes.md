# mise includes — what was checked

Verified with mise 2026.10.9 against a local git server, and against the mise
and Renovate sources. These are the constraints devkit's layout is built on.

## Two include mechanisms, two files per brick

| | `include = [...]` (top level) | `task_config.includes = [...]` |
|---|---|---|
| Carries | `[tools]`, `[env]`, `[vars]`, `[hooks]`, `[shell_alias]`, `[plugins]`… | tasks (TOML file or directory of file tasks) |
| Refuses | tasks, `task_templates`, `task_config`, `[settings]`, nested `include` | `task_templates` |
| `git::` | `https`/`ssh` only | `https`/`http`/`ssh`, **needs `experimental = true`** |
| Precedence | the including file wins; a later entry beats an earlier one | **the last entry wins**, so local tasks go *after* `git::` entries; tasks declared inline in `mise.toml` always win |
| Lockfile | tools are locked in the including file's `mise.lock` | — |

Hence `mise/<brick>/mise.toml` (tools/env, via `include`) and
`mise/<brick>/tasks.toml` (tasks, via `task_config.includes`).

Consequences:

- Tool versions shared across tasks cannot come from `task_templates` in a
  shared file (rejected in both include kinds). They come from the brick's
  `[tools]`, which also puts them in the shell and the IDE.
- Setting `task_config.includes` replaces the default task directories:
  `mise-tasks` is listed explicitly (a missing directory is fine).
- A wildcard dependency matching no task is an error (`depends = ["test:*"]`
  with no `test:` task), so the generated `mise.toml` only declares the
  aggregates its stacks can fill.
- `run` strings are Tera templates: bash `${#array[@]}` opens a Tera comment and breaks the task, and Go templates (kubeconform's
  `{{.Group}}`) must sit inside `{% raw %}…{% endraw %}`.
- Remote task repos are cached in `MISE_CACHE_DIR/remote-git-tasks-cache`;
  always pin a tag or commit.

## Renovate

- Recent Renovate only (main as of 2026-10; 42.x does not read `include` yet): the mise manager reads `[tools]`, `tasks.*.tools`, `tasks.*.file` and the
  `git::` entries of `include` (looked up as git tags), in files named like a
  mise config (`mise.toml`, `.mise.toml`, `mise/config.toml`…). devkit's bricks
  are therefore named `mise.toml`.
- It does **not** read `task_config.includes`: the devkit default preset adds a
  regex manager for `git::…//…tasks*.toml?ref=…` entries.
- It never fetches included files, so tools move only when the devkit tag does.

## Authentication

mise fetches `git::` includes with a built-in git client (gitoxide), not the
`git` binary. Checked against an HTTPS server requiring basic auth:

- no credentials: fails with "Could not obtain identity";
- `url.<with-token>.insteadOf` in the global git config, the
  `GIT_CONFIG_COUNT`/`GIT_CONFIG_KEY_n` environment form, and a credential
  helper (`credential.helper store`) all work;
- it refuses to send credentials over plain `http://`;
- the cached clone keeps the original URL: the token is not written to
  `MISE_CACHE_DIR`, which CI caches between jobs.

The GitLab components rely on the `insteadOf` form with `CI_JOB_TOKEN` (or
`DEVKIT_GIT_TOKEN`); the GitHub workflows with an optional `git-token` secret.
