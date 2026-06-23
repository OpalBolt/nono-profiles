# nono-profiles

nono sandbox profiles for AI coding agents (pi, Claude Code, GitHub Copilot).

## Structure

```
meta-base           extends linux-host-compat
├── pi-base         extends meta-base
│   ├── pi          extends pi-base
│   ├── pi-go       extends pi-base + go_runtime
│   ├── pi-node     extends pi-base + node_runtime
│   └── pi-python   extends pi-base + python_runtime
├── claude-base     extends meta-base
│   ├── claude      extends claude-base
│   ├── claude-go   extends claude-base + go_runtime
│   ├── claude-node extends claude-base + node_runtime
│   └── claude-python extends claude-base + python_runtime
└── copilot-base    extends meta-base
    ├── copilot     extends copilot-base
    ├── copilot-go  extends copilot-base + go_runtime
    ├── copilot-node extends copilot-base + node_runtime
    └── copilot-python extends copilot-base + python_runtime
```

`meta-base` handles shared concerns — filesystem access, network, git auth. Agent `<name>-base` profiles add agent-specific paths and security settings. `-go`/`-python`/`-node` variants include the matching runtime group.

## Git authentication

You keep your SSH remotes (`git@github.com:...`). Inside the sandbox, git
automatically rewrites them to HTTPS and authenticates with a PAT — so `~/.ssh`
stays blocked.

### How it works

The auth flow uses git's own `GIT_CONFIG_COUNT` / `GIT_CONFIG_KEY_N` / `GIT_CONFIG_VALUE_N`
environment variables — a git feature since v1.7.12 (2012). nono's only role is `set_vars`,
which puts them into the sandbox environment. Git does the rest.

**Step by step:**

1. **envoke** sets `GH_TOKEN` in your shell.
2. **nono** starts the sandbox — `allow_vars` lets `GH_TOKEN` through, `set_vars` injects
   `GIT_CONFIG_*` variables.
3. **Git** reads `GIT_CONFIG_*` and treats them as in-memory `.gitconfig` entries:
   - `url.*.insteadOf` rewrites `git@github.com:` to `https://github.com/`
   - `credential.*.helper` provides a command to fetch the token
4. **`git push`** triggers the credential helper (`gh`, `printf`), which reads the
   token from the environment and returns it to git.
5. **Git authenticates** with the token — no SSH keys, no files on disk.

| Host | Token var | Helper |
|---|---|---|
| GitHub | `GH_TOKEN` | `gh auth git-credential` |
| GitLab | `GL_TOKEN` | basic auth via printf |
| Codeberg | `CB_TOKEN` | basic auth via printf |

If GitLab or Codeberg push auth breaks, swap to their native CLIs — same pattern as
GitHub: `!glab auth git-credential` or `!tea auth git-credential`.

### Usage

1. Install [envoke](https://github.com/OpalBolt/envoke) and configure your PATs.
2. Run `envoke resolve` before launching a session.
3. Start your agent — `nono run pi-go`.

Git push/pull uses the PAT automatically. SSH remotes are untouched outside the sandbox.

Fine-grained PATs need `Contents: Read and write` and must be authorized for the target repo.

## Filesystem grants

- `/tmp` — read+write (compilers, shells, temp files)
- `~/.local/share/rtk/` — read+write (app data)
- `~/.config/gh` — read (GitHub CLI config)
- `~/.cache/nix` — read (nix eval cache)
- `$WORKDIR` — read+write (project directory)

`~/.ssh` is blocked by the `deny_credentials` group — SSH keys never enter the sandbox.

## Adding a new host

Add rows to `meta-base.json` `set_vars`, increment `GIT_CONFIG_COUNT`, and add the token to `allow_vars`. For self-hosted instances, swap the domain.
