# GoodLinks2Insta - sync saved links from GoodLinks to Instapaper

GoodLinks2Insta automatically syncs saved links from GoodLinks to your Instapaper account. This tool bridges the gap between GoodLinks (Apple-optimized) and Instapaper (better integration ecosystem).

This is an unofficial app and not endorsed by GoodLinks or Instapaper.

## Installation

```bash
# Clone the repository
git clone https://github.com/samplereality/goodlinks-to-instapaper.git
cd goodlinks-to-instapaper

# Install as a uv tool
uv tool install .
```

After pulling changes, reinstall with `uv tool install --force --reinstall .` (without `--reinstall`, uv may keep using a cached build).

After installation, you can run `goodlinks2insta` from anywhere:

```bash
goodlinks2insta init
goodlinks2insta sync
goodlinks2insta status
```

## Requirements

- macOS
- [uv](https://github.com/astral-sh/uv)
- GoodLinks app
- Instapaper account

## Installing uv on macOS

If you don't have uv installed, you can install it with:

```bash
# Using Homebrew (recommended)
brew install uv

# Or using the official installer
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Verify the installation:
```bash
uv --version
```

## Setup

```bash
# Initialize config with your Instapaper credentials
goodlinks2insta init
```

This creates a config at `~/Library/Application Support/goodlinks2insta/config.json`.

## Usage

### Typical commands

```bash
goodlinks2insta sync --dry-run   # preview: list what would be sent, send nothing
goodlinks2insta sync             # send new unread links to Instapaper
goodlinks2insta status           # counts, state file location, next links pending
goodlinks2insta sync --quiet     # for scheduled runs: log to file only
```

Running `goodlinks2insta` with no command is the same as `goodlinks2insta sync`.

Starting over (for example after clearing Instapaper), with a list of URLs that are already there:

```bash
goodlinks2insta reset
goodlinks2insta sync --dry-run --skip-urls instapaper-urls.txt   # check first
goodlinks2insta sync --skip-urls instapaper-urls.txt
```

### Commands

| Command | What it does |
|---|---|
| `init` | Prompts for your Instapaper email and password and writes the config file. `--force` / `-f` overwrites an existing config. |
| `sync` | Sends new links to Instapaper (the default command). Takes the flags below. |
| `status` | Shows how many links GoodLinks has, how many are recorded as synced, where the state file is, and the first pending links. |
| `reset` | Empties the sync record, so the next `sync` treats every eligible link as new. |
| `version` | Prints the version (same as `goodlinks2insta --version` or `-v`). |

### `sync` flags

Flags apply to a single run only; they never change the config file.

| Flag | What it does | Config equivalent |
|---|---|---|
| `-n`, `--dry-run` | List what would be sent and skipped, then stop. Sends nothing and doesn't update the state file. | none |
| `-q`, `--quiet` | No console output; messages go only to the log file. Use for scheduled runs. | none |
| `--include-read` | Also send links already marked read in GoodLinks. | `"include_read": true` turns this on for every run |
| `--skip-urls FILE` | FILE lists URLs already in Instapaper, one per line (`#` starts a comment). GoodLinks links with matching URLs are recorded as synced without being sent. Matching ignores `http`/`https`, `www.`, query strings and trailing slashes. | none |
| `--allow-full-sync` | Send even when no state file exists at all (normally refused above 25 links, as a safeguard). Not needed after `reset`. | none |
| `-r N`, `--max-retries N` | Retries per link on network or server errors (default 3). | none |

## How flags and the config file relate

- The **config file** holds settings that stay the same from run to run: your Instapaper login, whether to open GoodLinks, where to log, where the state file lives, and whether read links are included.
- **Flags** change one run. The only setting that exists in both places is read links: `--include-read` turns it on for one run even when the config says `"include_read": false`. (No flag turns it off when the config has it on.)
- The **state file** (`synced.json`) is neither: it's the tool's record of which GoodLinks links have been sent. `sync` adds to it, `reset` empties it, and `--dry-run` leaves it alone.

## Background Sync (every 3 hours)

To set up automatic syncing every 3 hours, the plist file is already in your cloned repository:

### 1. Edit the plist file (if needed)

Open `com.sidget.goodlinks2insta.plist` (it's already in your cloned repository) and configure the paths:

- The UV path is already set to `/opt/homebrew/bin/uv` for Homebrew installations
  - If you installed uv elsewhere, replace this with the output from `which uv`
- The command is already set to `goodlinks2insta` (the installed uv tool)

### 2. Install the launch agent

```bash
cp com.sidget.goodlinks2insta.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.sidget.goodlinks2insta.plist
```

**The plist file is already configured for most Homebrew users:**
```xml
<array>
    <string>/opt/homebrew/bin/uv</string>
    <string>tool</string>
    <string>run</string>
    <string>goodlinks2insta</string>
    <string>sync</string>
    <string>--quiet</string>
</array>
```

If you're not using Homebrew, update the UV path accordingly before copying the file.

Logs: `~/Library/Logs/goodlinks2insta.log`

To uninstall the background sync:
```bash
launchctl unload ~/Library/LaunchAgents/com.sidget.goodlinks2insta.plist
rm ~/Library/LaunchAgents/com.sidget.goodlinks2insta.plist
```

**Important**: Use the uv tool command structure:
`/opt/homebrew/bin/uv tool run goodlinks2insta sync --quiet`

Since `goodlinks2insta` is installed as a uv tool, it's available on your PATH and can be called directly from anywhere.

## What gets synced

- Only links that are **unread** in GoodLinks are sent (set `"include_read": true` in the config, or pass `--include-read`, to send everything).
- Links are sent **oldest-saved first**, about one per second, so the newest GoodLinks saves end up at the top of Instapaper.
- Duplicate URLs in GoodLinks are sent once.
- `--skip-urls FILE` takes a file of URLs (one per line) that are already in Instapaper; matching links are recorded as synced without being sent.

## Sharing the sync record between Macs

The sync record (`synced.json`) lists every GoodLinks link that has been sent to Instapaper (or deliberately skipped). If you sync from more than one Mac, every Mac has to read **the same** record, or each one will re-send links the others already sent.

`goodlinks2insta status` shows which record a Mac is using on its `State:` line. Check that it's the same path on every Mac.

### Where the record lives

1. **`state_file` in the config**, if set. Use this to put the record in any folder that syncs between your Macs, such as Dropbox:

   ```json
   "state_file": "~/Library/CloudStorage/Dropbox/goodlinks2insta/synced.json"
   ```

   Set the same `state_file` on every Mac. Check the folder name with `ls ~/Library/CloudStorage/` first; with more than one Dropbox account it may be `Dropbox-Personal` or similar, and older Dropbox installs use `~/Dropbox`.
2. **Otherwise, iCloud Drive**: `iCloud Drive/goodlinks2insta/synced.json`, if iCloud Drive is turned on for that Mac.
3. **Otherwise, a local file** at `~/Library/Application Support/goodlinks2insta/synced.json`. This record is **not shared**, and the tool prints a warning on every run. That happens on Macs where iCloud Drive is off, for example a work Mac managed by an IT department; use `state_file` there instead.

### Moving to a shared record

When a Mac first uses a shared location that doesn't have a record yet, its local `~/Library/Application Support/goodlinks2insta/synced.json` is copied there and the local file is renamed `synced.json.migrated`, so it can't be used by mistake later.

To set up a second Mac, or switch both Macs to Dropbox:

1. On the Mac with the up-to-date record, copy it to the shared folder (find its current path on the `State:` line of `goodlinks2insta status`).
2. On every Mac, set `state_file` in the config to that shared path.
3. On every Mac, run `goodlinks2insta status`. The `State:`, `Synced:` and `Pending:` lines should match. Don't run `sync` on a Mac until they do.

Do step 1 before setting `state_file` on any other Mac. Otherwise that Mac may copy its own, possibly out-of-date, record into the shared folder first.

### Safeguards

- If iCloud has offloaded the record to save space, the tool downloads it before syncing.
- If there's no record at all and more than 25 links would be sent, `sync` refuses, so a new or misconfigured Mac can't flood Instapaper. Run `goodlinks2insta reset` or pass `--allow-full-sync` when a full sync is intended.
- `reset` writes an empty record rather than deleting it, so the reset reaches every Mac.
- When saving, the tool merges its list with what's already in the record, so a Mac doesn't erase entries another Mac added.

Dropbox and iCloud take a little while to copy changes between Macs, so schedule automatic syncs on only one Mac. Running `sync` by hand on another Mac is then safe.

## Config

`goodlinks2insta init` creates `~/Library/Application Support/goodlinks2insta/config.json`. Edit it to change these settings:

```json
{
  "username": "your@email.com",
  "password": "...",
  "launch_goodlinks": true,
  "sync_delay": 5,
  "include_read": false,
  "log_file": "~/Library/Logs/goodlinks2insta.log",
  "state_file": "~/Library/CloudStorage/Dropbox/goodlinks2insta/synced.json"
}
```

| Key | Default | What it does |
|---|---|---|
| `username`, `password` | (set by `init`) | Your Instapaper login. Required. |
| `launch_goodlinks` | `true` | Open GoodLinks in the background if it isn't running (and quit it afterward). If `false`, the sync is skipped when GoodLinks isn't open. |
| `sync_delay` | `5` | Seconds to wait after launching GoodLinks so it can pull new links from iCloud. |
| `include_read` | `false` | Also send links already marked read in GoodLinks. |
| `log_file` | `~/Library/Logs/goodlinks2insta.log` | Where every run is logged, including `--quiet` runs. |
| `state_file` | iCloud Drive `goodlinks2insta/synced.json`, or `~/Library/Application Support/goodlinks2insta/synced.json` (not shared) when iCloud Drive is off | Where the record of sent links is kept. Set the same path on every Mac; see [Sharing the sync record between Macs](#sharing-the-sync-record-between-macs). |

Only `username` and `password` are required; leave out any other key to use its default. The config file is per Mac and isn't shared, so run `init` and set `state_file` on each Mac.

## Uninstalling the tool

If you installed with `uv tool install`:

```bash
uv tool uninstall goodlinks2insta
```
