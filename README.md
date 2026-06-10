# flamp

**agentic engineering with wings**

`flamp` runs an Amp prompt on a disposable Fly Machine, commits the result, and pushes it back to your GitHub branch. Launch your agent, close your laptop, get a push notification to your phone when it's done.

## Requirements

- A Git repository with an `origin` remote, preferably on GitHub.
- The [Fly CLI](https://fly.io/docs/flyctl/) installed and authenticated.
- The [Amp CLI](https://ampcode.com/) installed locally. `flamp` uses it to generate `Dockerfile.amp` when one does not already exist.
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

Extra path or glob arguments are copied into the cloned repository on the Fly Machine at the same relative path. They are added to `.git/info/exclude` there, so they are available to Amp but are not committed.

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
2. Generates `Dockerfile.amp` if the repo does not already have one.
3. Starts a disposable Fly Machine with the repo, prompt, SSH key, and optional extra files.
4. Clones the GitHub repository, checks out or creates the requested branch, and runs Amp in deep mode.
5. Commits any changes, pulls the latest branch state, asks Amp to resolve merge conflicts if needed, and pushes back to `origin`.

## Notes

- The Fly Machine receives the SSH key as a mounted file and uses it only for Git operations.
- Generated or supplied `Dockerfile.amp` controls the cloud development environment.
- Extra local files are intentionally excluded from commits on the remote Machine.
