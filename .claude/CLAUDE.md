# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A GNOME Shell extension (uuid: `claude-code-usage@haletran.com`) that reads Claude Code's OAuth credentials from `~/.claude/.credentials.json`, calls the Anthropic usage API (`https://api.anthropic.com/api/oauth/usage`), and renders 5-hour and 7-day utilization as a panel indicator with an optional progress bar.

## Installation & development workflow

There is no build step for JavaScript. The only compile step is the GSettings schema:

```bash
cd schemas && glib-compile-schemas .
```

To install/reload the extension during development, run the `update` script from the repo root's **parent** directory (it `cd ..` internally):

```bash
./claude-usage-extension/update
# This copies the extension to ~/.local/share/gnome-shell/extensions/claude-code-usage@haletran.com/,
# recompiles the schema, then calls gnome-session-quit --no-prompt to restart the session.
```

After a GNOME Shell restart, re-enable the extension if needed:

```bash
gnome-extensions enable claude-code-usage@haletran.com
```

To view runtime logs (extension errors, API failures):

```bash
journalctl -f -o cat /usr/bin/gnome-shell
```

## Architecture

The extension has two entry points required by GNOME Shell:

- **`extension.js`** — loaded when the extension is enabled. Exports `ClaudeUsageExtension` (subclass of `Extension`). Its `enable()` creates a `ClaudeUsageIndicator` and adds it to `Main.panel`; `disable()` destroys it.
- **`prefs.js`** — loaded when the user opens Settings. Exports `ClaudeUsagePreferences` (subclass of `ExtensionPreferences`). Builds an Adwaita preferences window with `Adw.PreferencesPage/Group/Row` widgets.

`ClaudeUsageIndicator` (in `extension.js`) is the core widget — a `PanelMenu.Button` subclass that:
1. Reads `~/.claude/.credentials.json` (or `$CLAUDE_CONFIG_DIR/.credentials.json`) to extract `claudeAiOauth.accessToken`.
2. Fetches `GET https://api.anthropic.com/api/oauth/usage` with `Authorization: Bearer <token>` and `anthropic-beta: oauth-2025-04-20`.
3. Parses `data.five_hour.utilization` and `data.seven_day.utilization` (0–100 floats).
4. Updates panel label/progress bar and dropdown menu items on each refresh tick.

**Settings** (`schemas/org.gnome.shell.extensions.claude-code-usage.gschema.xml`) define five keys: `refresh-interval` (int, default 300s), `display-mode` (string: `text`/`bar`/`both`), `icon-style` (string: `color`/`monochrome`), `show-icon` (bool), `proxy-url` (string). Changes are observed via `settings.connect('changed', ...)` in `_init`.

**Styling** lives entirely in `stylesheet.css`. Progress bar color is driven by CSS class names (`usage-low/medium/high/critical`) applied dynamically based on thresholds (40/70/90%).

## Key constraints

- All GNOME Shell APIs (`St`, `Clutter`, `GLib`, `Gio`, `Soup`) are imported via the `gi://` URI scheme — standard Node.js/browser APIs are unavailable.
- The extension must be installed to the exact path `~/.local/share/gnome-shell/extensions/claude-code-usage@haletran.com/` to be recognized by GNOME Shell.
- Schema changes require rerunning `glib-compile-schemas .` in the `schemas/` directory and restarting the shell.
- Soup async callbacks run on the GLib main loop; never block the main thread.
