# flamp

**agentic engineering with wings**

`flamp` runs an [Amp](https://ampcode.com) prompt on a disposable [Fly](http://fly.io/) Machine, commits the result, and pushes it back to your GitHub branch. Launch your agent, close your laptop, get a push notification to your phone when it's done.

## Requirements

- A Git repository with an `origin` remote, preferably on GitHub.
- The [Fly CLI](https://fly.io/docs/flyctl/) installed and authenticated.
- The [Amp CLI](https://ampcode.com/) installed locally. `flamp` uses it to generate `flamp-deps.Dockerfile` (the project's base image + dependency-install steps) when one does not already exist.
- An Amp API key in `AMP_API_KEY`.
- An SSH private key with push access to the repo in `CLOUD_SSH_KEY_PATH`.

## Usage

```bash
export AMP_API_KEY=...
export CLOUD_SSH_KEY_PATH=~/.ssh/id_ed25519

./flamp <branch> "your prompt here" [path-or-glob ...]
./flamp <branch> @prompt.md [path-or-glob ...]
./flamp <branch> - [path-or-glob ...]
cat prompt.md | ./flamp <branch>
```

Examples:

```bash
./flamp fix/readme "Add a README for this project"
./flamp debug-run @prompt.md logs/latest.log .env.local
cat prompt.md | ./flamp feature/new-api
```

Pass `HEAD` as the branch to use your currently checked-out branch:

```bash
./flamp HEAD @prompt.md
```

The Docker image is built from a clean checkout of your current local commit (`HEAD`), not your local worktree — and it is pinned to that commit rather than re-fetching `origin`'s default-branch tip on every run, so the Docker layer cache stays warm across runs. (To force a refresh of the baked dependencies, delete `Dockerfile.amp` / `flamp-deps.Dockerfile` and re-run.) The baked checkout only seeds the image's dependency layers; at runtime the Fly Machine fetches and checks out the requested branch, so the branch (including `HEAD`) must already be pushed with the commits you want Amp to work from — uncommitted local changes are not picked up.

Extra path or glob arguments are copied into the checked-out repository on the Fly Machine at the same relative path. They are added to `.git/info/exclude` there, so they are available to Amp but are not committed.

## Configuration

Environment variable | Default | Description
--- | --- | ---
`AMP_API_KEY` | required | Amp API key from ampcode.com/settings.
`CLOUD_SSH_KEY_PATH` | required | SSH private key with push access to the repository.
`FLY_APP` | `amp-runner` | Fly app used to launch the Machine.
`FLY_REGION` | `ord` | Fly region for the Machine.
`FLY_VM_SIZE` | `shared-cpu-2x` | Fly Machine CPU size.
`FLY_VM_MEMORY` | `2048` | Fly Machine memory in MB.
`FLAMP_SIMPLEPUSH_KEY` | unset | Optional Simplepush key for Amp's final output.

(`ord` is the closest fly region to ampcode.com)

## How it works

1. Reads the prompt from an argument, `@file`, `-`, or stdin.
2. Builds a Docker context from a clean checkout of your current local commit (`HEAD`), pinned to that commit (not `origin`'s latest default-branch tip), excluding local worktree files.
3. Assembles the Dockerfile from a single-stage fragment, `flamp-deps.Dockerfile` (the base image + system/dependency steps, which Amp generates — reusing the repo's existing Dockerfile where possible — if the repo does not already have one). flamp injects the repo worktree (`WORKDIR /tmp/home/work` + `COPY . /tmp/home/work`, **without** `.git`) right after the fragment's `FROM`, then appends git/ssh/ripgrep and the Amp CLI install. `.git` is deliberately excluded: a fresh clone's `.git` is not byte-stable, so baking it in would bust the Docker layer cache on every build (the runtime fetch recreates `.git` on the Machine). Supplying your own complete `Dockerfile.amp` overrides this entirely.
4. Starts a disposable Fly Machine with the baked-in repo, prompt, SSH key, and optional extra files.
5. Fetches and checks out or creates the requested branch in the baked-in repo, then runs Amp in deep mode.
6. Commits any changes, pulls the latest branch state, asks Amp to resolve merge conflicts if needed, and pushes back to `origin`.

## Notes

- The Fly Machine receives the SSH key as a mounted file and uses it only for Git operations.
- `flamp-deps.Dockerfile` is a single-stage Dockerfile fragment: its `FROM` plus the system/dependency `RUN`/`ENV` lines, ideally lifted from the repo's own Dockerfile. flamp injects the repo checkout right after the `FROM` and appends the tooling/Amp install, so the steps the runner depends on stay pinned in flamp's code rather than the agent's. It must contain exactly one `FROM`; for multi-stage or fully custom builds, use a complete `Dockerfile.amp` instead.
- For full control over the image, commit your own `Dockerfile.amp`. When present it is built verbatim and must copy the build context into `/tmp/home/work`, install dependencies, and install the Amp CLI itself. The build context no longer contains `.git`; the runner initializes git at runtime, so you don't need to bake it in (if you do, the runner reuses it).
- Extra local files are intentionally excluded from commits on the remote Machine.
