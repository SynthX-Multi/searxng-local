# Self-Hosted SearXNG (Docker) — Private, Local, Long-Term Search Engine

A ready-to-run, locally hosted [SearXNG](https://github.com/searxng/searxng) setup using Docker Compose. SearXNG is a free metasearch engine that queries many search engines at once and shows you the combined results without tracking you or building a profile.

This repo is tuned for **personal, long-term use on one machine** (especially Windows 10/11 with Docker Desktop), with:

- Access limited to your own computer (`127.0.0.1`)
- Containers that restart automatically
- Memory limits so it doesn't hog your RAM
- Valkey (a Redis-compatible store) for SearXNG's short-term data
- Image proxying, so result images load through your instance and not directly from third parties
- The Mojeek engine enabled as an independent-index option
- A full troubleshooting section covering the problems you're most likely to hit

---

## Table of Contents

1. [Features](#features)
2. [What's in this repo](#whats-in-this-repo)
3. [Requirements](#requirements)
4. [Quick start (Windows)](#quick-start-windows)
5. [Quick start (Linux / macOS)](#quick-start-linux--macos)
6. [Configuration](#configuration)
7. [Use it as your default search engine](#use-it-as-your-default-search-engine)
8. [Keep it running long term](#keep-it-running-long-term)
9. [Reducing memory usage](#reducing-memory-usage)
10. [Updating](#updating)
11. [Backup and restore](#backup-and-restore)
12. [Troubleshooting](#troubleshooting)
13. [Security notes](#security-notes)
14. [Uninstalling everything](#uninstalling-everything)
15. [Useful commands cheat sheet](#useful-commands-cheat-sheet)
16. [Credits and license](#credits-and-license)

---

## Features

| Feature | Details |
|---|---|
| Local-only access | Port is bound to `127.0.0.1:8080`, so other devices on your network can't reach it |
| Auto-restart | `restart: unless-stopped` on both services |
| Memory caps | `512m` for SearXNG, `128m` for Valkey |
| Persistent data | Named Docker volumes for the cache and Valkey data |
| Config outside the container | Your settings live in `./core-config/` on your PC |
| Image proxy | `image_proxy: true`, so image results are fetched by your instance |
| Favicons | Resolved via DuckDuckGo |
| Extra engine | Mojeek enabled (see the [CPU warning](#enabling-mojeek)) |

---

## What's in this repo

```
.
├── docker-compose.yml     # The two services: SearXNG (core) and Valkey
├── .env.example           # Template for environment variables
├── .env                   # Your local copy (created by you, not committed)
├── core-config/
│   └── settings.yml       # Your SearXNG settings (created by you, not committed)
├── .gitignore             # Keeps secrets and personal data out of git
└── README.md
```

### `docker-compose.yml` at a glance

- **`core`** is the SearXNG app, published only on `127.0.0.1:8080`. It mounts `./core-config/` at `/etc/searxng/` and has a named volume for its cache.
- **`valkey`** is the data store SearXNG uses for short-term state. It saves snapshots to its own volume.
- Both services have `restart: unless-stopped` and a `mem_limit`.
- The image tag comes from `SEARXNG_VERSION` in `.env` and defaults to `latest`.

---

## Requirements

- **Docker** with Docker Compose v2 (the `docker compose` command, no hyphen)
  - Windows/macOS: [Docker Desktop](https://www.docker.com/products/docker-desktop/)
  - Linux: Docker Engine plus the Compose plugin
- About **1 GB of free RAM** for the Docker VM on Windows/macOS, plus SearXNG itself
- Internet access, since SearXNG queries upstream search engines for you
- **Windows only:** virtualization must be enabled in your BIOS/UEFI. Check in Task Manager → Performance → CPU → "Virtualization: Enabled". Windows 10 needs version 2004 or later, and Windows 11 is fine. Home edition works.

---

## Quick start (Windows)

> **Important:** the default Windows PowerShell (5.1) is *not* the same as a Linux shell.
> - `&&` doesn't work, so run commands one line at a time.
> - `curl` is an alias for a different command (`Invoke-WebRequest`), so Linux `curl -fsSLO ...` commands will fail.
> - Paths containing spaces must be wrapped in quotes.

### 1. Install WSL2 (without Ubuntu)

Docker Desktop needs WSL2 as its engine backend, but you **don't** need to install Ubuntu or any Linux distribution. This avoids the Ubuntu "create a username" prompt entirely.

Open PowerShell **as administrator** (right-click Start → Terminal (Admin)):

```powershell
wsl --install --no-distribution
```

Restart your PC when it finishes.

### 2. Install Docker Desktop

In a normal PowerShell window:

```powershell
winget install Docker.DockerDesktop
```

Then:

1. Restart if prompted.
2. Open **Docker Desktop** from the Start menu and accept the license. You can skip signing in.
3. Wait until the engine shows as running (green, bottom left).
4. In **Settings → General**, make sure **Use the WSL 2 based engine** is ticked, and tick **Start Docker Desktop when you sign in**.

Verify:

```powershell
docker --version
docker compose version
```

> Docker Desktop is free for personal use and small businesses. Larger organizations need a paid subscription.

### 3. Get the project

```powershell
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>
```

No git? Download the repo as a ZIP from GitHub, extract it, and `cd` into the extracted folder. For example:

```powershell
cd "C:\path\with spaces\to\SearXNG"
```

### 4. Create your `.env`

```powershell
copy .env.example .env
```

The defaults are fine. Uncomment and edit lines in `.env` only if you want to pin a version or change the listen address/port.

### 5. Create your settings file

Create `core-config\settings.yml` (make the `core-config` folder if it doesn't exist):

```powershell
mkdir core-config
notepad core-config\settings.yml
```

Paste this, then replace the secret key (next step):

```yaml
use_default_settings: true

server:
  secret_key: "CHANGE_ME"
  image_proxy: true

search:
  favicon_resolver: "duckduckgo"

engines:
  - name: mojeek
    inactive: false
    disabled: false
```

Generate a strong secret key in PowerShell (works on Windows PowerShell 5.1):

```powershell
$b = New-Object byte[] 32; [Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b); ($b | ForEach-Object { $_.ToString("x2") }) -join ""
```

Copy the 64-character output into `secret_key`, replacing `CHANGE_ME`. Never share or commit it.

> **Why not `Get-Random`?** A common one-liner such as `-join ((48..57)+(97..102) | Get-Random -Count 64 ...)` doesn't give you 64 random characters. With a 16-item source, `-Count 64` returns only 16 characters, each used once, which makes a weak key. Use the .NET method above instead.

### 6. Start it

```powershell
docker compose up -d
```

The first run downloads the images, so it can take a few minutes. Check status:

```powershell
docker compose ps
```

Open **http://localhost:8080**. If you see the SearXNG search page, you're done.

---

## Quick start (Linux / macOS)

**macOS:** install Docker Desktop (`brew install --cask docker`) and open it once so the engine starts.
**Linux:** install Docker Engine and the Compose plugin (for example, `curl -fsSL https://get.docker.com | sudo sh`), then optionally run `sudo usermod -aG docker $USER` and log out and back in.

```bash
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>

cp .env.example .env
mkdir -p core-config

# Generate a strong secret key
openssl rand -hex 32
```

Create `core-config/settings.yml` using the YAML from [step 5 above](#5-create-your-settings-file), paste the generated key into `secret_key`, then:

```bash
docker compose up -d
docker compose ps
```

Open **http://localhost:8080**.

---

## Configuration

### Environment variables (`.env`)

| Variable | Purpose | Default |
|---|---|---|
| `SEARXNG_VERSION` | Image tag, for example `latest` or a specific release tag | `latest` |
| `SEARXNG_HOST` | Address SearXNG listens on inside the container | `[::]` |
| `SEARXNG_PORT` | Port SearXNG listens on inside the container | `8080` |

More options are in the official docs for [general settings](https://docs.searxng.org/admin/settings/settings_general.html) and [server settings](https://docs.searxng.org/admin/settings/settings_server.html).

### `core-config/settings.yml`

Because `use_default_settings: true` is set, you only list what you want to **change**. Everything else comes from SearXNG's built-in defaults.

```yaml
use_default_settings: true

server:
  secret_key: "YOUR_GENERATED_KEY"   # required: unique and private
  image_proxy: true                   # load result images through your instance

search:
  favicon_resolver: "duckduckgo"      # show site icons in results

engines:
  - name: mojeek
    inactive: false                   # lift the built-in "inactive" flag
    disabled: false                   # turn the engine on
```

**YAML rules to avoid errors:**
- Use **spaces only**, never tabs. Indent with 2 spaces.
- Keep only **one** `engines:` section and one `use_default_settings:` line. Duplicate keys break YAML, and one silently wins.
- Save as `settings.yml`, not `settings.yml.txt` (Windows Notepad can do this by accident).

After editing, apply changes with:

```bash
docker compose restart core
```

### Enabling Mojeek

Mojeek is an engine with its own independent index, which makes it a useful alternative to the big engines. SearXNG ships it **turned off in two ways**:

| Flag | Meaning |
|---|---|
| `inactive: true` | The engine isn't loaded at all, so it won't even appear in Preferences. |
| `disabled: true` | The engine is loaded but switched off by default. |

Setting only `disabled: false` is **not enough** because the built-in `inactive: true` still applies. You need both `inactive: false` and `disabled: false`, as in the config above.

> **CPU warning:** SearXNG marks Mojeek inactive because it uses a Proof-of-Work CAPTCHA, which makes your machine do real computing work for each query. On a laptop or low-power PC you may see brief CPU spikes or slower searches. If that bothers you, set `inactive: true` again and restart.

`mojeek images` and `mojeek news` are separate engines with the same restriction. Enable them the same way if you want them:

```yaml
  - name: mojeek images
    inactive: false
    disabled: false
```

### Optional: enable the JSON API

Only needed if you want to use your instance from scripts or other apps (for example, AI tools or Open WebUI). Add to `settings.yml`:

```yaml
search:
  formats:
    - html
    - json
```

Then `docker compose restart core`.

---

## Use it as your default search engine

Add a custom search engine in your browser with this URL (`%s` is where your query goes):

```
http://localhost:8080/search?q=%s
```

- **Chrome / Edge:** Settings → Search engine → Manage search engines and site search → Add.
- **Firefox:** open `http://localhost:8080`, right-click the address bar, and choose **Add "SearXNG"** (or add it in Settings → Search).
- **Brave / others:** use the custom search engine section in settings with the URL above.

Then set it as the default.

> Docker Desktop must be running for this to work. If your search engine "doesn't load" after a reboot, give it about a minute for Docker and the containers to start.

---

## Keep it running long term

1. **Docker Desktop starts at login:** Settings → General → *Start Docker Desktop when you sign in*.
2. **Containers restart automatically:** already configured with `restart: unless-stopped`.
3. **Keep it local-only:** the compose file binds to `127.0.0.1`. Don't change this unless you understand the [security implications](#security-notes).
4. **Update regularly:** see [Updating](#updating). Updating also keeps the engine definitions fresh, which matters because upstream search engines change often.
5. **Back up your config:** see [Backup and restore](#backup-and-restore).

---

## Reducing memory usage

On Windows and macOS, Docker runs inside a lightweight VM. In Task Manager you'll see it as **`vmmemWSL`** (Windows) and it often shows about 1 GB. Much of that is the Linux file cache, not SearXNG itself.

### 1. See what's actually using memory

```bash
docker stats --no-stream
```

### 2. Limit the WSL2 VM (Windows)

Create `C:\Users\<you>\.wslconfig`. **It must be in your user folder** (`%USERPROFILE%`); WSL ignores it anywhere else.

```powershell
Set-Content -Path "$env:USERPROFILE\.wslconfig" -Value @"
[wsl2]
memory=1536MB
processors=2
swap=0

[experimental]
autoMemoryReclaim=gradual
sparseVhd=true
"@
```

| Setting | Effect |
|---|---|
| `memory=1536MB` | Hard ceiling for the VM. Don't go much below 1 GB. |
| `processors=2` | Limits CPU cores the VM can use. |
| `swap=0` | Disables the swap file. |
| `autoMemoryReclaim=gradual` | Returns unused memory to Windows over time (needs a recent WSL version). |
| `sparseVhd=true` | Lets the virtual disk shrink as you free space. |

If WSL complains about unknown settings, delete the `[experimental]` section.

Apply it: quit Docker Desktop (tray icon → Quit), then:

```powershell
wsl --shutdown
```

Reopen Docker Desktop.

### 3. Container memory caps (already set)

The compose file limits SearXNG to `512m` and Valkey to `128m`. If a container keeps restarting, it may be hitting its cap, so raise the limit and run `docker compose up -d` again.

### 4. Clear short-term data and reset memory growth

SearXNG has no big cache to clear. These steps are safe:

```bash
docker compose exec valkey valkey-cli FLUSHALL
docker compose restart
```

### 5. Drop the Linux file cache (temporary)

```powershell
wsl -d docker-desktop sh -c "echo 3 > /proc/sys/vm/drop_caches"
```

### 6. Give everything back

When you're not searching:

```bash
docker compose down
```

Then quit Docker Desktop and run `wsl --shutdown`. `vmmemWSL` disappears from Task Manager.

### Other ways to save resources

- Disable engines you never use (fewer parallel requests per search).
- Keep Mojeek off if CPU or memory is tight.
- Turn off **Autocomplete** in Preferences → General to avoid request bursts.
- Docker Desktop's **Resource Saver** mode can pause the engine when idle.

---

## Updating

### Update SearXNG

```bash
docker compose down
docker compose pull
docker compose up -d
```

Your settings and data are preserved.

### Update Docker Desktop

- **In the app:** open Docker Desktop and click the update notice, then **Update and restart**.
- **With winget:** quit Docker Desktop first, then:

```powershell
winget upgrade Docker.DockerDesktop
```

After updating, check that everything came back:

```bash
docker --version
docker compose ps
```

If the containers aren't running, `docker compose up -d` starts them. Back up your config first (see below), though updates don't touch it.

---

## Backup and restore

The files that matter:

- `.env`
- `docker-compose.yml`
- `core-config/` (especially `settings.yml`)

Copy the whole project folder somewhere safe. To restore, put the folder back and run `docker compose up -d`.

Named volumes (`core-data`, `valkey-data`) hold only caches and short-term state, so you don't need to back them up.

---

## Troubleshooting

### "Cannot connect to the Docker daemon" / Docker commands fail

Docker Desktop isn't running. Open it and wait for the engine to show green.

### PowerShell errors: `The token '&&' is not a valid statement separator`

Windows PowerShell 5.1 doesn't support `&&`. Run commands one per line.

### `curl: A parameter cannot be found that matches parameter name 'fsSLO'`

In PowerShell `curl` is an alias for `Invoke-WebRequest`. Use the Linux commands from a Linux shell, or the PowerShell form:

```powershell
Invoke-WebRequest <url> -OutFile <filename>
```

### Windows asks to "set default user for Ubuntu" or WSL asks to be installed again

You don't need Ubuntu. Use `wsl --install --no-distribution` (see [Quick start](#1-install-wsl2-without-ubuntu)). If a `wsl` command shows *"Windows Subsystem for Linux must be updated... Press any key to install"*, press **ESC** to cancel. It just means WSL isn't installed (or you removed it), and it shows up when you run any `wsl` command.

### The page won't load at `localhost:8080`

```bash
docker compose ps
docker compose logs core --tail 60
```

- Make sure both services show **Up**.
- **Port already in use:** change the left side of the port mapping in `docker-compose.yml`, for example `"127.0.0.1:8081:8080"`, run `docker compose up -d`, and browse to `localhost:8081`.

### Mojeek doesn't appear in Preferences → Engines

The built-in definition has `inactive: true`, and `disabled: false` alone doesn't override it. Your `settings.yml` needs both:

```yaml
engines:
  - name: mojeek
    inactive: false
    disabled: false
```

Then `docker compose restart core`. Also:
- Look under **Engines → General** (Mojeek is a general web engine). Use an incognito window, since browser-saved preferences can hide changes.
- Verify the container sees your edit:
  ```bash
  docker compose exec core grep -n -B1 -A3 mojeek /etc/searxng/settings.yml
  ```
- Test directly: `http://localhost:8080/search?q=test&engines=mojeek`
- Check what the built-in default says:
  ```bash
  docker compose exec core grep -n -B2 -A8 "name: mojeek" /usr/local/searxng/searx/settings.yml
  ```

### Brave (or other engines) keep failing: "Too many requests"

You'll see lines like `SearxEngineTooManyRequestsException ... suspended_time=180` in the logs. This means the upstream engine is **rate-limiting your IP**. Unlike Mojeek's Proof-of-Work, there's no puzzle to compute past, so SearXNG just suspends that engine for a few minutes and retries. This is normal on home connections. Ways to reduce it:

- Disable engines you don't use, since every search hits every enabled engine.
- Turn off **Autocomplete** in Preferences → General.
- Rely on several engines (DuckDuckGo, Startpage, Mojeek, etc.) so one suspension doesn't matter.
- Use a different network or IP if blocks persist.

### Log messages that are harmless for local use

| Message | Why you can ignore it |
|---|---|
| `missing favicon config: /etc/searxng/favicons.toml` | Optional file |
| `missing config file: /etc/searxng/limiter.toml` | Optional file |
| `X-Forwarded-For nor X-Real-IP header is set!` | Only matters behind a reverse proxy |
| `ahmia` / `torch` "can't register engine" | Tor-only engines you aren't using |
| `engine timeout` for individual engines | Upstream slowness |
| `ResourceWarning: unclosed database` | Python housekeeping noise |

### Config changes don't apply

- Run `docker compose restart core` after every `settings.yml` edit.
- Check the file the container actually uses:
  ```bash
  docker compose config
  ```
  Look for the bind mount whose target is `/etc/searxng`. That source folder is the one to edit.
- Check logs for YAML errors: `docker compose logs core --tail 60`.

### A container keeps restarting

It may be hitting its memory cap. Check `docker stats --no-stream`, raise `mem_limit` in `docker-compose.yml`, and run `docker compose up -d`.

---

## Security notes

- **Do not commit secrets or personal data.** Never put `.env`, `core-config/settings.yml` (it contains your `secret_key`), browser cookies, exported data files, or logs in a public repo. Use the [`.gitignore`](#gitignore) below.
- **Keep it bound to localhost.** `127.0.0.1:8080:8080` means only your own machine can connect. If you change it to `8080:8080` or `0.0.0.0`, anyone on your network can use your instance.
- **Never expose it directly to the internet.** If you want remote access, put it behind a reverse proxy with HTTPS and authentication, and enable SearXNG's [limiter/bot detection](https://docs.searxng.org/admin/searx.botdetection.html), which expects the proxy to set `X-Forwarded-For` / `X-Real-IP`. A VPN to your home network is a simpler option.
- **Rotate the secret key if it ever leaks.** Generate a new one, update `settings.yml`, and restart. You'll lose saved preferences and cookies, which is harmless.
- **Privacy has limits.** SearXNG hides your identity from the engines it queries, but those engines still see your server's IP and the search terms. It's not an anonymity tool like Tor.

### `.gitignore`

```gitignore
# Secrets and local config
.env
core-config/settings.yml
core-config/*.toml

# Personal data, cookies, exports, logs
data.json
*.cookies
cookies*.txt
*.log

# OS / editor junk
.DS_Store
Thumbs.db
.vscode/
```

If your repo should include a template, commit a sanitized `core-config/settings.yml.example` with `secret_key: "CHANGE_ME"`, and let users copy it to `settings.yml`.

---

## Uninstalling everything

### Just SearXNG

```bash
docker compose down -v --rmi all
```

Then delete the project folder.

### Docker, WSL and all leftovers (Windows)

Run each command separately in PowerShell.

1. **Remove SearXNG and all Docker data:**
   ```powershell
   cd <your-project-folder>
   docker compose down -v --rmi all
   docker system prune -a --volumes
   ```
   `prune` removes *everything* unused in Docker, not just this project, so skip it if you have other containers.
2. **Delete the project folder**, keeping a backup of `.env` and `core-config` if you want them.
3. **Quit Docker Desktop**, then uninstall it:
   ```powershell
   winget uninstall Docker.DockerDesktop
   ```
4. **Remove Docker's WSL data:**
   ```powershell
   wsl --list --verbose
   wsl --unregister docker-desktop
   ```
   If you get the "WSL must be updated" prompt, press **ESC**, since there's nothing to remove.
5. **Uninstall WSL:**
   ```powershell
   wsl --uninstall
   ```
6. **Turn off Windows features:** search *Turn Windows features on or off* and untick **Virtual Machine Platform** and **Windows Subsystem for Linux**, then restart.
7. **Delete leftover folders** (ignore "not found" errors):
   ```powershell
   Remove-Item -Recurse -Force $env:AppData\Docker
   Remove-Item -Recurse -Force "$env:AppData\Docker Desktop"
   Remove-Item -Recurse -Force $env:LocalAppData\Docker
   Remove-Item -Recurse -Force C:\ProgramData\DockerDesktop
   Remove-Item -Recurse -Force ~\.docker
   Remove-Item ~\.wslconfig
   ```
8. **Remove the search engine** from your browser's settings, then **restart your PC**.

Verify with `docker --version`, which should now say "not recognized". Avoid running `wsl` commands afterwards, since they just show the install prompt.

---

## Useful commands cheat sheet

```bash
docker compose up -d                 # start (or apply compose changes)
docker compose ps                    # status
docker compose logs -f core          # follow SearXNG logs (Ctrl+C to exit)
docker compose logs core --tail 60   # last 60 log lines
docker compose restart core          # apply settings.yml changes
docker compose restart               # restart everything
docker compose down                  # stop and remove containers
docker compose pull                  # download newer images
docker compose config                # show the final merged config (check mounts)
docker stats --no-stream             # per-container memory/CPU
docker compose exec valkey valkey-cli FLUSHALL   # clear short-term data
```

Windows reminders: one command per line, quote paths with spaces, and open Docker Desktop first.

---

## Credits and license

- [SearXNG](https://github.com/searxng/searxng): the metasearch engine this project runs. SearXNG is licensed under AGPL-3.0.
- [Valkey](https://valkey.io/): the in-memory data store.
- [Docker](https://www.docker.com/) and [Docker Compose](https://docs.docker.com/compose/).
- Official SearXNG Docker docs: <https://docs.searxng.org/admin/installation-docker.html>

Add your own license for the files in this repository (for example MIT) in a `LICENSE` file. Search results come from third-party engines, and their terms apply to your use of them.
