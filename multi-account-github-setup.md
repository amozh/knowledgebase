# Managing Multiple GitHub Accounts on One Machine (macOS)

A setup for using two GitHub accounts (e.g., work and personal) on the same machine with zero manual switching. Git and `gh` CLI automatically use the correct account based on which directory you're in.

## How It Works

- Projects live in two separate directories (e.g., `~/work/` and `~/personal/`)
- Each directory is associated with a GitHub account
- SSH keys, git identity, and `gh` CLI tokens auto-select based on the project location
- No manual switching needed — just clone into the right folder

## Prerequisites

- macOS with Homebrew
- `git` installed
- `gh` CLI installed (`brew install gh`)

## Step 1: Generate SSH Keys

Create a separate key for each account:

```bash
ssh-keygen -t ed25519 -C "work@example.com" -f ~/.ssh/id_work
ssh-keygen -t ed25519 -C "personal@example.com" -f ~/.ssh/id_personal
```

Add both keys to the SSH agent with Keychain persistence:

```bash
ssh-add --apple-use-keychain ~/.ssh/id_work
ssh-add --apple-use-keychain ~/.ssh/id_personal
```

## Step 2: Upload Public Keys to GitHub

Add each public key to the corresponding GitHub account:

- `~/.ssh/id_work.pub` → GitHub Settings > SSH keys (work account)
- `~/.ssh/id_personal.pub` → GitHub Settings > SSH keys (personal account)

## Step 3: SSH Config

Create/edit `~/.ssh/config`:

```
Host github-work
  HostName github.com
  User git
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_work
  IdentitiesOnly yes

Host github-personal
  HostName github.com
  User git
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_personal
  IdentitiesOnly yes
```

`IdentitiesOnly yes` is critical — it prevents SSH from trying other keys from the agent.

Verify:

```bash
ssh -T github-work      # Should print: Hi <work-username>!
ssh -T github-personal   # Should print: Hi <personal-username>!
```

## Step 4: Git Config with Directory-Based Switching

### `~/.gitconfig`

```ini
[core]
    autocrlf = input

[user]
    name = <work-username>
    email = <work-email>

[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work

[includeIf "gitdir:~/personal/"]
    path = ~/.gitconfig-personal
```

The default identity is set to work. The `includeIf` directives override settings based on the repo's location.

### `~/.gitconfig-work`

```ini
[url "git@github-work:"]
    insteadOf = git@github.com:
    insteadOf = https://github.com/
```

### `~/.gitconfig-personal`

```ini
[user]
    name = <personal-username>
    email = <personal-email>

[url "git@github-personal:"]
    insteadOf = git@github.com:
    insteadOf = https://github.com/
```

The `insteadOf` rules transparently rewrite any `github.com` URL to the correct SSH host alias, so both HTTPS and SSH clone URLs work automatically.

**Important:** Do NOT put `insteadOf` rules in the base `~/.gitconfig`. They must be in the directory-specific include files to avoid conflicts.

Verify:

```bash
# From a work repo:
git -C ~/work/<some-repo> config user.email        # work email
git -C ~/work/<some-repo> ls-remote --get-url origin  # git@github-work:...

# From a personal repo:
git -C ~/personal/<some-repo> config user.email          # personal email
git -C ~/personal/<some-repo> ls-remote --get-url origin  # git@github-personal:...
```

## Step 5: `gh` CLI with Auto-Switching via direnv

### Install and configure direnv

```bash
brew install direnv
```

Add to `~/.zshrc` (or `~/.bashrc`):

```bash
# direnv
export DIRENV_LOG_FORMAT=""
eval "$(direnv hook zsh)"  # use "bash" for bash
```

### Silence direnv output

direnv 2.36.0+ has a [known bug](https://github.com/direnv/direnv/issues/1418) where `DIRENV_LOG_FORMAT=""` is ignored unless a config file exists. Create `~/.config/direnv/direnv.toml`:

```toml
[global]
log_filter = "^$"
hide_env_diff = true
```

### Log in to both GitHub accounts

```bash
gh auth login  # log in as work account
gh auth login  # log in as personal account
```

Verify both are registered:

```bash
gh auth status
# Should list both accounts
```

### Create `.envrc` files

In the work directory root (e.g., `~/work/.envrc`):

```bash
export GH_TOKEN=$(gh auth token --user <work-username>)
```

In the personal directory root (e.g., `~/personal/.envrc`):

```bash
export GH_TOKEN=$(gh auth token --user <personal-username>)
```

Allow both:

```bash
direnv allow ~/work/.envrc
direnv allow ~/personal/.envrc
```

Tokens are not stored on disk — `gh auth token` pulls them dynamically from macOS Keychain at each shell session.

Verify:

```bash
cd ~/work && gh api user --jq '.login'      # work username
cd ~/personal && gh api user --jq '.login'   # personal username
```

## Summary

| Component | Work directory | Personal directory |
|---|---|---|
| SSH key | `~/.ssh/id_work` | `~/.ssh/id_personal` |
| Git user | work username + email | personal username + email |
| URL rewrite | `git@github-work:` | `git@github-personal:` |
| `gh` CLI | work account token | personal account token |

Everything auto-selects based on directory. Clone into the right folder and never think about it again.

## Troubleshooting

**SSH authenticates as wrong account**: Check that `IdentitiesOnly yes` is set in `~/.ssh/config`. Clear the SSH agent (`ssh-add -D`) and re-add only your two keys.

**`insteadOf` rewrites to wrong host**: Make sure `insteadOf` rules are NOT in the base `~/.gitconfig` — they must only be in the directory-specific include files (`~/.gitconfig-work`, `~/.gitconfig-personal`).

**`gh` uses wrong account**: Run `echo $GH_TOKEN` to check if direnv set it. If empty, verify the `.envrc` is allowed (`direnv allow`).

**direnv still shows log messages**: Ensure `~/.config/direnv/direnv.toml` exists with `log_filter = "^$"`. The env var alone is not enough in direnv 2.36.0+.
