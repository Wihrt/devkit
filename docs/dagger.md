# Dagger 1.0 with mise — an experiment

Status: **experiment**, written against Dagger `v1.0.0-beta.21` (October 2026)
from its source and docs. Nothing here is part of a devkit release contract yet.

## The split

devkit's rule stays: **mise owns the tooling and the tasks**. Dagger only
replaces what GitLab CI and GitHub Actions do today — deciding where and how a
`mise run <task>` executes — and does it better on three points:

| | GitLab CI / GitHub Actions | Dagger |
|---|---|---|
| Where a task runs | one job = one VM/container, YAML per platform | one container per task, same on a laptop and in CI |
| Cache | `.mise/` cache keyed on `mise.toml`, upload/download per job | content-addressed layers: an unchanged task does not run again |
| Parallelism | jobs, stages, `needs:` | every check of `dagger check` runs in parallel |
| Local reproduction | `mise run lint` (same tools, not the same isolation) | `dagger check`, byte-for-byte the CI run |

What Dagger usually also does — bring the tools through its official modules
(`dagger.io/go`, `dagger.io/python/pytest`, …), each with its own versions — is
deliberately **not** used: tool versions would then live in two places. The
`dagger/mise` module installs nothing itself; it runs `mise install` and
`mise run` in a container.

```
 host                              Dagger engine (containers)
 ────                              ─────────────────────────────────────────────
 mise.toml ── dagger = 1.0.0-beta  toolchain: jdxcode/mise image
 mise run dagger:check               + config files only (mise.toml, mise.lock…)
   └─ dagger check ───────────────►  + `mise install`          ◄─ cached layer
        dagger.toml                runner: toolchain + project at /src (+ .git)
          [modules.mise]             ├─ mise run lint:prek     ┐
                                     ├─ mise run lint:secrets  ├ one check each,
                                     └─ mise run lint:actions  ┘ in parallel
```

## What is in this repository

| Path | Role |
|---|---|
| `dagger/mise/` | The Dagger module, in [Dang](https://docs.dagger.io/reference/sdks/dang) (`main.dang` + `dagger-module.toml`, no codegen, nothing generated to commit) |
| `mise/dagger/mise.toml` | New brick: pins the `dagger` CLI (aqua backend) |
| `mise/dagger/tasks.toml` | `dagger:check`, `dagger:list`, `dagger:fmt` — host tasks driving Dagger |
| `dagger.toml` | devkit's own Dagger workspace: installs `dagger/mise` from the working tree |
| `.github/workflows/dagger.yml` | Runs devkit's lint tasks through Dagger, next to `ci.yml` |

## The module

`dagger/mise/main.dang` exposes:

- **`tasks`** — a [collection](https://docs.dagger.io/reference/sdks/dang#collections)
  of the project's mise tasks whose name starts with a `checkPrefixes` entry
  (default `lint:`, `test:`). Each item has a `run` check. Aggregates such as
  `lint` (which only `depends` on `lint:*`) are left out on purpose: each leaf
  task is its own check, so Dagger parallelises and caches them one by one.
- **`fmt`** — a generator (`@generate`): runs the `fmt:*` tasks and returns a
  changeset. `dagger generate` shows it for review; `dagger check` turns it into
  a "stale" check that fails when the code is not formatted.
- **`run(task)`**, **`build(task, output)`**, **`shell`** — plain functions for
  everything else (`dagger api call mise …`).

Two containers are built:

1. **toolchain** — the `jdxcode/mise` image (same tag as the GitLab components)
   with only the mise config files mounted, then `mise install`. Its cache key
   is those files: editing code never reinstalls a tool, editing `mise.toml`
   always does.
2. **runner** — toolchain + the whole workspace at `/src`, and the git history
   at `/src/.git` (`Workspace.git.directory`, needed by gitleaks and prek).

One cache volume (`/cache`, via `XDG_CACHE_HOME`) holds mise downloads, the uv,
Go and golangci-lint caches: they survive runs without being part of a layer.

### Settings

Constructor arguments are settings, set in `dagger.toml` (or
`dagger module settings mise <key> <value>`):

| Setting | Default | Purpose |
|---|---|---|
| `checkPrefixes` | `["lint:", "test:"]` | Which tasks become checks |
| `fmtPrefixes` | `["fmt:"]` | Which tasks the `fmt` generator runs |
| `variables` | `[]` | `KEY=VALUE` for every task, e.g. `["UV_LOCKED=1"]` |
| `dir` | `.` | Directory holding `mise.toml` (monorepo) |
| `exclude` | `.mise`, `.venv`, `dist`, … | Paths kept out of the source (cache churn) |
| `git` | `true` | Mount the history at `.git`. Ties every check to the commit: turn it off when no task reads git |
| `image` | devkit's `jdxcode/mise` tag | Base image (mise + git + curl only) |
| `githubToken` | — | Secret for mise's GitHub API calls, e.g. `"env://GITHUB_TOKEN"` |

`UV_LOCKED=1` illustrates a difference with the CI wrappers: they set strict
variables only in CI, because local and CI runs differ. Under Dagger both are
the same run, so the strict value can simply be the setting.

## Try it on devkit

```sh
mise install                 # now also installs the dagger CLI
mise run dagger:list         # = dagger check -l --all
mise run dagger:check        # = dagger check   (needs Docker or Podman)
dagger check --mise-task=lint:actions
dagger api call mise shell terminal
```

devkit sets `checkPrefixes = ["lint:"]`: its only test task, `test:template`,
renders the Copier template against a commit pushed to GitHub, and stays in
the `ci.yml` matrix.

## Use it in a project

```sh
dagger init
dagger module install github.com/Wihrt/devkit/dagger/mise@X.Y.Z
dagger check -l --all
```

and add the brick to `mise.toml` so the CLI version is pinned with the rest:

```toml
include = [
  "git::https://github.com/Wihrt/devkit.git//mise/dagger/mise.toml?ref=X.Y.Z",
  # …the other bricks
]

[task_config]
includes = [
  "git::https://github.com/Wihrt/devkit.git//mise/dagger/tasks.toml?ref=X.Y.Z",
  # …
  "mise-tasks",
]
```

### CI

The CI job shrinks to "have Docker, have mise, run one task".

GitHub Actions (see `.github/workflows/dagger.yml`):

```yaml
- uses: actions/checkout@v7
  with: { fetch-depth: 0 }
- uses: jdx/mise-action@v5
- run: mise run dagger:check
  env:
    DAGGER_ENV: ci
    GITHUB_TOKEN: ${{ github.token }}
```

GitLab CI needs a container runtime for the engine: Docker-in-Docker, or —
better, since the Dagger cache lives in the engine — a long-lived engine the
job connects to with `DAGGER_ENGINE` (for example `container://…`, `ssh://…`,
or the engines of the official Helm chart):

```yaml
dagger:
  image: <an image with mise and the docker CLI>   # the engine starts through docker
  services: [docker:dind]
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
    DAGGER_ENV: ci
  script:
    - mise install dagger
    - mise run dagger:check
```

Or skip the CI file altogether: `dagger cloud checks on` runs the same checks
on Dagger Cloud after each push.

## What changes compared with the components and reusable workflows

- **One file instead of two per platform**: `dagger.toml` replaces
  `templates/<brick>/template.yml` and `.github/workflows/<brick>.yml` for the
  lint/test part. The tasks themselves do not move.
- **Tasks are discovered, not listed**: a new `lint:foo` task in any brick or
  in `mise-tasks/` becomes a check without touching CI.
- **Caching is per task, by content**: a commit touching only a Helm chart
  re-runs only the tasks whose inputs changed — as long as `git = false`.
  With the history mounted, every commit changes the inputs.
- **Granularity is the task**: Dagger cannot see inside `mise run test:go`;
  it caches and reports the task as a whole. The official `dagger.io/go`
  module goes down to one check per test, at the price of owning the Go
  toolchain.
- **Artifacts and publishing** stay plain functions (`build`, `run`):
  `dagger api call mise build --output=./dist`. Registry credentials would be
  `Secret` settings, never plain env vars.

## Not verified yet

This was written without running Dagger: the sandbox it was written in could
not reach `registry.dagger.io`. The first `dagger.yml` run is the real test.
Points to watch:

- Dang syntax of `main.dang` (checked against the docs and the modules shipped
  in the Dagger repository, not against the parser).
- `mise tasks ls --no-header` output (first column = task name).
- The `dagger` aqua package for a `1.0.0-beta.N` version.
- `Workspace.git.directory` on a GitHub Actions checkout.
- Renovate: the `dagger` CLI (mise manager) and `engineVersion` (inline hint)
  come as separate updates and must stay in step.

## References

- [Dagger 1.0 beta announcement](https://dagger.io/blog/dagger-10-beta/)
- Dagger docs: [Dang SDK](https://docs.dagger.io/reference/sdks/dang),
  [checks](https://docs.dagger.io/using/checking),
  [`dagger.toml`](https://docs.dagger.io/reference/config-files/dagger-toml),
  [includes](https://docs.dagger.io/config/includes)
