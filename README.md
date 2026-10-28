# devkit

A standard base for any project: **mise** brings the tools and the tasks,
**GitLab CI** and **GitHub Actions** only orchestrate, **prek** runs the git
hooks, **Renovate** keeps everything current, and **Copier** stamps it all
into a project, then keeps it up to date.

```
            ┌──────────── devkit (one tag: X.Y.Z) ────────────┐
            │ mise/<brick>/mise.toml   tools + env             │
            │ mise/<brick>/tasks.toml  lint:* test:* build:* … │
            │ templates/<brick>/       GitLab CI components    │
            │ .github/workflows/       GitHub reusable wfs     │
            │ renovate/                Renovate presets        │
            │ template/ + copier.yml   project template        │
            └──────────────────────────────────────────────────┘
                 │ include git::…?ref=X.Y.Z      │ component …@X.Y.Z / uses …@X.Y.Z
                 ▼                               ▼
      your project: mise.toml ── mise run lint|test|build ◄── CI job = `mise run <task>`
```

The rule that holds it together: **a CI job never contains logic, it runs
`mise run <task>`**. What CI runs is what you run locally, with the same tool
versions, and switching between GitLab and GitHub changes nothing but the
orchestration.

Bricks available today: `base` (always), `python`, `go`, `helm`, and the
experimental `dagger` brick ([Dagger 1.0 with mise](docs/dagger.md)).

## Start a project

```sh
mise use -g copier          # or: uv tool install copier
copier copy gh:Wihrt/devkit my-project
cd my-project && mise trust && mise install
mise run                    # pick a task
```

Copier asks for the stacks (several for a monorepo), the CI orchestrator(s)
and whether to generate a minimal skeleton. On an existing project, answer
"no" to the skeleton: only the tooling files are written.

Later, follow a new devkit release with `copier update`: it bumps every
reference (mise includes, components, workflows) to the new tag.

### What gets generated

| File | Role |
|---|---|
| `mise.toml` | `include` of the bricks' tools/env, `task_config.includes` of their tasks, `lint`/`fmt`/`test`/`build` aggregates |
| `.pre-commit-config.yaml` | prek hooks: hygiene, gitleaks, ruff / golangci-lint fmt / helm-docs — through `mise exec`, so the versions are mise's |
| `.gitlab-ci.yml` | one `include: component:` per brick |
| `.github/workflows/ci.yml` | one reusable workflow per brick |
| `renovate.json` | `extends` the devkit presets |
| skeleton (optional) | `pyproject.toml` + `src/` + `tests/`, `go.mod` + `main.go`, `charts/<name>/` |

Git hooks install themselves: the base brick declares a mise `enter` hook
running `prek install` (or run `mise run setup:hooks`).

## Tasks

| Brick | Tasks | Tools (pinned) |
|---|---|---|
| base | `lint:prek`, `lint:secrets`, `lint:actions`, `setup:hooks` | prek, gitleaks, actionlint |
| python | `install:python`, `lint:python`, `fmt:python`, `test:python`, `build:python` | uv, ruff (uv brings Python) |
| go | `lint:go`, `fmt:go`, `test:go`, `build:go` | go, golangci-lint |
| helm | `deps:helm`, `lint:helm`, `test:helm` (helm template → kubeconform), `docs:helm`, `build:helm`, `publish:helm` | helm, kubeconform, helm-docs |
| dagger (experimental) | `dagger:check`, `dagger:list`, `dagger:fmt` — run the other bricks' `lint:*`/`test:*` tasks in Dagger containers | dagger |

Differences between local and CI are explicit variables set by the CI
wrappers, never a test on `$CI` inside a task: `UV_LOCKED=1` (stale `uv.lock`
fails), `HELM_REGISTRY` / `HELM_REGISTRY_USERNAME` / `HELM_REGISTRY_PASSWORD`,
`GO_TEST_FLAGS`, `KUBERNETES_VERSION`, `CHARTS_DIR`, `HELM_DIST_DIR`.

### Overriding

Your `mise.toml` always wins over devkit:

```toml
[tools]
ruff = "0.16.0"            # another version than devkit's

[env]
CHARTS_DIR = "deploy/charts"

[tasks."test:python"]      # replace a shared task
run = "uv run pytest -n auto"
```

Local file tasks go in `mise-tasks/`, listed after devkit's includes, so they
win too.

## Use the pieces à la carte

**mise only**

```toml
include = ["git::https://github.com/Wihrt/devkit.git//mise/go/mise.toml?ref=X.Y.Z"]

[settings]
experimental = true   # git:: task includes are experimental

[task_config]
includes = ["git::https://github.com/Wihrt/devkit.git//mise/go/tasks.toml?ref=X.Y.Z", "mise-tasks"]
```

**GitLab CI**

```yaml
stages: [lint, test, build, publish]
include:
  - component: gitlab.com/wihrt/devkit/base@X.Y.Z
  - component: gitlab.com/wihrt/devkit/go@X.Y.Z
  - component: gitlab.com/wihrt/devkit/mise@X.Y.Z   # any task, as one job
    inputs: { job_prefix: e2e, task: "test:e2e", stage: test }
```

Every component takes `job_prefix` (change it when a component is included
twice, otherwise jobs silently overwrite each other), `root_dir` (monorepo),
`git_auth` (see [private repositories](#private-repositories-who-clones-the-includes))
and `image` (default `jdxcode/mise:<version>-debian`). Add a masked `GITHUB_TOKEN`
CI/CD variable (a fine-grained token with no permission is enough) to avoid
GitHub rate limits when mise downloads tools. The `.mise/` directory is cached.

**GitHub Actions**

```yaml
jobs:
  go:
    uses: Wihrt/devkit/.github/workflows/go.yml@X.Y.Z
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: Wihrt/devkit@X.Y.Z      # composite action: any task
        with: { task: "test:e2e" }
```

**Renovate**

```json
{ "extends": ["github>Wihrt/devkit//renovate/default", "github>Wihrt/devkit//renovate/go"] }
```

Presets are read from `main` without a version — `main` is protected and
reviewed. The default preset groups every devkit reference (mise includes,
task includes, components, workflows) into a single "devkit" PR.

## Monorepo

Pick several stacks, or include a component once per sub-project with its own
`root_dir` and `job_prefix`. For larger monorepos, route from the root
`.gitlab-ci.yml` to child pipelines (`trigger:` + `rules:changes` with
`compare_to` + `strategy: depend`), each sub-directory carrying its own
`.gitlab-ci.yml` and `mise.toml`. Selectivity in CI comes from
`rules:changes` / `paths:`, not from mise `sources`/`outputs` (those help
locally).

## Versioning and releases

One tag for everything (`X.Y.Z`, no `v`: the GitLab CI/CD Catalog wants plain
semver). Conventional commits on `main` feed release-please; a tool bump in a
brick is a `fix`, so it ships as a patch.

Two Renovate levels:

1. on devkit, Renovate bumps the tools pinned in `mise/*/mise.toml`, the CI
   image and the actions;
2. on your projects, Renovate bumps the devkit tag (Renovate does not read the
   included files, so tools move only through devkit releases).

## Private repositories: who clones the includes?

mise clones its `git::` includes itself, inside the job, **before** any task
runs. With devkit public on GitHub nothing is needed. With a private copy (a
devkit mirror on your GitLab, private task repositories), the job needs an
identity, and git `insteadOf` gives it one without touching any URL in
`mise.toml` — mise honours it, as well as credential helpers (checked against
an HTTPS server requiring auth; the token never lands in the mise cache).

**GitLab** — every component rewrites `https://$CI_SERVER_FQDN/` to carry a
token (`git_auth: job_token`, the default):

- by default `CI_JOB_TOKEN`. On the **included** project (the devkit mirror,
  each private task repo): *Settings → CI/CD → Job token permissions →
  Authorized groups and projects*, add the consuming projects — adding their
  top-level group is a one-off. The job token carries the permissions of the
  user who runs the pipeline, who must be able to read that project.
- or a CI/CD variable `DEVKIT_GIT_TOKEN` (group access token or deploy token
  with `read_repository`, plus `DEVKIT_GIT_USER` for a deploy token's
  username), set once on the group: no allowlist to maintain, one secret to
  rotate.
- `git_auth: none` sends no credentials. Use it when everything included is
  public and the project is not in the allowlist: GitLab rejects a job token
  outside the allowlist even for a public project.

`include: component:` is checked separately: the user running the pipeline
must be able to read the components project.

**GitHub** — `github.token` only reads the current repository. Pass a token
with read access to the private repositories as the `git-token` secret of the
reusable workflows (or input of the action); the generated `ci.yml` forwards a
`DEVKIT_GIT_TOKEN` repository/organization secret when it exists.

**Locally** — your usual git credentials (credential helper, `glab auth
setup-git`, `gh auth setup-git`) are used.

**Renovate** needs read access too: a `hostRules` entry for your GitLab host
to look up devkit tags.

## GitLab CI/CD Catalog

Components must live in a GitLab project of the instance that includes them.
devkit is developed here and mirrored to GitLab:

1. Create the GitLab project (e.g. `gitlab.com/wihrt/devkit`), set a
   description, and turn on *Settings → General → Visibility → CI/CD Catalog
   project*.
2. Create a project access token (role Maintainer, scope `write_repository`).
3. On GitHub, add the repository variable `GITLAB_MIRROR`
   (`gitlab.com/wihrt/devkit.git`) and the secret `GITLAB_TOKEN`.

`mirror-gitlab.yml` then pushes `main` and tags, the release workflow pushes
release tags, and the GitLab pipeline publishes each tag to the catalog. A
self-managed GitLab gets its own mirror the same way.

## Developing devkit

```sh
mise install
mise run lint
mise run test:template python,go,helm   # render + lint/test/build the result
```

`test:template` includes the bricks by `git::` URL, so it tests a pushed
commit (`DEVKIT_REF`, default `HEAD`; `DEVKIT_REPO` to point elsewhere). CI
runs it for every stack.

[docs/dagger.md](docs/dagger.md) shows Dagger 1.0 as the executor of the
mise tasks, mise keeping the tooling.

Further notes: [docs/mise-includes.md](docs/mise-includes.md) records how mise
includes actually behave, checked against mise 2026.10.

## Roadmap

kustomize brick, container image build (`BUILDER`, `IMAGE_CACHE_REF`), Java,
PHP, Python type checking.
