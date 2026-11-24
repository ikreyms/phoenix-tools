# phoenix-tools

One command, `phoenix`, to set up and run the Phoenix development environment.

```bash
phoenix setup     # clone all repos into a new project folder
phoenix up        # start containers and install dependencies
phoenix update    # update these tools
```

## Requirements

- Linux, macOS, or Windows through WSL2 (run everything inside the WSL2 shell)
- `git` and `docker` with Compose v2 (`docker compose version`)
- A GitHub SSH key with access to `ikreyms/phoenix-tools` and the `aasandhadev` organisation. Test with `ssh -T git@github.com`.

## Install

Run these as your normal user. Do not start with `sudo` (see [Using sudo](#using-sudo)).

### Standard install

```bash
git clone git@github.com:ikreyms/phoenix-tools.git ~/.phoenix-tools
bash ~/.phoenix-tools/install
```

This puts the tools in `~/.phoenix-tools` and links `phoenix` into `~/.local/bin`.

### With the GitHub CLI

Needs [`gh`](https://cli.github.com/) installed and logged in:

```bash
gh api repos/ikreyms/phoenix-tools/contents/install \
  -H "Accept: application/vnd.github.raw" | bash
```

### Custom locations

Two environment variables change where things go:

| Variable | Default | Meaning |
|---|---|---|
| `PHOENIX_HOME` | `~/.phoenix-tools` | where the repo is cloned |
| `PHOENIX_BIN_DIR` | `~/.local/bin` | where the `phoenix` link is created |

Locations you own, no sudo needed. Always run the `install` file by its full path:

```bash
# Different link folder (after the standard clone)
PHOENIX_BIN_DIR=~/bin bash ~/.phoenix-tools/install

# Different tools folder: clone there yourself, then point PHOENIX_HOME at it
git clone git@github.com:ikreyms/phoenix-tools.git ~/tools/phoenix
PHOENIX_HOME=~/tools/phoenix bash ~/tools/phoenix/install

# Both
PHOENIX_HOME=~/tools/phoenix PHOENIX_BIN_DIR=~/bin bash ~/tools/phoenix/install
```

With the CLI install, put the variables before `bash`:

```bash
gh api repos/ikreyms/phoenix-tools/contents/install \
  -H "Accept: application/vnd.github.raw" | PHOENIX_BIN_DIR=~/bin bash
```

### Using sudo

Install as yourself and use `sudo` only for the step that needs root. Running the whole installer as root makes git use root's SSH keys (the clone fails) and leaves root-owned files that `phoenix update` can't change.

**Link `phoenix` into `/usr/local/bin`** (a PATH folder that needs root):

```bash
bash ~/.phoenix-tools/install
sudo ln -sf ~/.phoenix-tools/phoenix /usr/local/bin/phoenix
```

Other users on the machine can only run this if they can read your home folder, so for a shared machine use the `/opt` layout below.

**Install the tools under `/opt`** (create the folder, give it to yourself, then clone into it):

```bash
sudo mkdir -p /opt/phoenix-tools
sudo chown "$USER" /opt/phoenix-tools
git clone git@github.com:ikreyms/phoenix-tools.git /opt/phoenix-tools
PHOENIX_HOME=/opt/phoenix-tools bash /opt/phoenix-tools/install
```

**Both, system-wide** (shared machine):

```bash
sudo mkdir -p /opt/phoenix-tools && sudo chown "$USER" /opt/phoenix-tools
git clone git@github.com:ikreyms/phoenix-tools.git /opt/phoenix-tools
PHOENIX_HOME=/opt/phoenix-tools bash /opt/phoenix-tools/install
sudo ln -sf /opt/phoenix-tools/phoenix /usr/local/bin/phoenix
```

**Already ran the installer with sudo?** Take ownership back:

```bash
sudo chown -R "$USER" ~/.phoenix-tools
```

### Add to PATH (if `phoenix` is not found)

```bash
# bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc

# zsh (macOS default)
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

Using a custom `PHOENIX_BIN_DIR`? Add that folder instead. Or skip PATH and call it by full path: `~/.phoenix-tools/phoenix up`.

### Check it works

```bash
phoenix help
```

### Uninstall

```bash
rm -f ~/.local/bin/phoenix
rm -rf ~/.phoenix-tools
sudo rm -f /usr/local/bin/phoenix    # only if you linked it there
```

Projects created with `phoenix setup` are not touched.

## The three commands

### `phoenix setup`: create a project

Clones the project and all its repos, then asks which branch to use for each app.

```bash
phoenix setup [-t parent_dir] [-d name]
```

| Option | Default | Meaning |
|---|---|---|
| `-t` | `/var/www` | parent directory (created if missing) |
| `-d` | `phoenix` | project folder name (a name only, no `/`) |

```bash
phoenix setup                      # /var/www/phoenix
phoenix setup -t ~/projects        # ~/projects/phoenix
phoenix setup -t . -d feenix       # ./feenix
```

- The project folder must not already exist. Nothing is overwritten.
- Ctrl+C or a failed clone removes the half-made project, so you can just re-run.
- At the branch prompts, press Enter to accept the default in brackets.
- If a checkout fails, setup finishes with a warning. Switch that repo's branch by hand.
- `/var/www` not writable? Use `-t ~/projects`, or run `sudo chown "$USER" /var/www` once.

### `phoenix up`: start everything

Starts the containers and installs dependencies in each app.

```bash
phoenix up [-t project_dir] [-b]
```

| Option | Default | Meaning |
|---|---|---|
| `-t` | current folder if it has `docker-compose.yml`, else `/var/www/phoenix` | project directory |
| `-b` | off | rebuild images with `--no-cache` first |

```bash
cd /var/www/phoenix && phoenix up    # current project
phoenix up -t ~/projects/phoenix     # a specific project
phoenix up -b                        # rebuild first
```

- The gulp and Vue watchers run in the background. Re-running does not start duplicates.
- Gate's Passport keys are generated only if `storage/oauth-private.key` is missing, so re-running does not log everyone out.
- When it finishes it prints a reminder of the manual steps: import the gate and phoenix databases, and set the `.env` files.
- Watch their logs:

  ```bash
  docker exec app_phoenix tail -f /tmp/gulp.log
  docker exec vue_phoenix tail -f /tmp/vue-watch.log
  ```

- Run it again any time (after a reboot, or after pulling changes).

### `phoenix update`: update the tools

```bash
phoenix update
```

Pulls the latest version. You can also re-run `bash ~/.phoenix-tools/install`, which additionally repairs the link and file permissions.

## Troubleshooting

| Problem | Fix |
|---|---|
| `phoenix: command not found` | Add the install folder to PATH (see above), then open a new terminal. |
| `Permission denied` running a script | `bash ~/.phoenix-tools/install` (repairs permissions). |
| `Permission denied (publickey)` | SSH key missing or no repo access. Check `ssh -T git@github.com`. |
| `already exists` during setup | Choose another `-d` or `-t`, or delete the old folder if it is disposable. |
| `Invalid directory name` | `-d` takes a name only. Use `-t . -d feenix`, not `-d ./feenix`. |
| `docker: 'compose' is not a docker command` | Install Docker Compose v2 or update Docker Desktop. |
| `No such container` | Containers are not running. Run `phoenix up` from inside the project. |
| `not a TTY` | Run `phoenix up` from a normal terminal, not cron or CI. |
