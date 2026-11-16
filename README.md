# CashCards LIVE Dashboard

CashCards LIVE Dashboard is a single-file desktop app (Python + Tkinter) that pulls TikTok Shop LIVE performance data and turns it into a "dead zone" dashboard. A dead zone is a stretch of a LIVE stream in which a chosen metric (items sold, orders, GMV or product clicks) stayed at zero for at least a set number of minutes.

The app exports per-session and per-minute LIVE data from the TikTok Shop Open API to CSV files. It then builds a self-contained HTML dashboard from the minute-level CSVs you pick and opens it in your browser. The dashboard works offline: its charting and CSV libraries and the data itself are embedded in the HTML file.

## Features

- **Guided flow.** Press **Start**, choose whether to export fresh data, pick the CSV files, and the dashboard opens.
- **TikTok Shop export**
  - Pulls LIVE sessions for both `OFFICIAL_ACCOUNTS` and `MARKETING_ACCOUNTS` over the last 1–90 days (default 7).
  - Fetches per-minute performance for each session, 6 sessions in parallel.
  - Retries with backoff on HTTP 429/5xx and on API rate-limit messages.
  - Caches minute data for LIVEs more than 2 days old, so repeat exports are faster.
  - If a CSV is locked (for example, open in Excel), saves it under a timestamped name instead.
- **One-time sign-in.** Opens the TikTok Shop authorization page and catches the code on a local callback (`http://localhost:8765/callback`). You can also paste a code or the full redirect URL by hand. The resulting tokens are saved and refreshed automatically.
- **Forgiving settings file.** Accepts `.env`, `env.txt` and similar names. Rich-text files saved by TextEdit (`.rtf` / `.rtfd`) are converted to a plain `env.txt`.
- **Dashboard (single HTML file)**
  - Metric picker: GMV, items sold, orders, customers, viewers, views, product clicks, new followers, likes, comments and others.
  - Dead-zone rule: no items sold, orders, GMV or product clicks for at least N minutes.
  - Filters: date range with presets (All, last 24h, 3 days, 7 days), time of day, day of week and account. You can keep or trim zero-viewer minutes at the start and end of a stream.
  - Views: KPIs, an over-time chart (total or per stream hour, by hour, day or week), insights, tables by account and host, a day-of-week by hour heatmap, a by-hour-of-day chart, a "minutes into the stream" chart, a dead-zone table (downloadable as CSV), a session list and a zoomable per-session drill-down.
  - Light/dark theme toggle and print / save as PDF.
  - You can add more CSVs inside the dashboard ("+ Add CSV" or drag and drop).

## Requirements

- **Prebuilt app:** Windows (`CashCards LIVE.exe`) or macOS (`CashCards LIVE.app.zip`). No Python needed.
- **From source:**
  - Python 3 with Tkinter. The python.org installers include Tkinter.
  - Standard library only, so there is nothing to `pip install`. `certifi` is used if present (see [Troubleshooting](#troubleshooting)).
- A TikTok Shop Partner app (app key and secret) authorized for the shop. Only the first authorized shop on the login is used.
- A web browser, for sign-in and for viewing the dashboard.
- For export only: network access to `open-api.tiktokglobalshop.com`, `auth.tiktok-shops.com` and `services.us.tiktokshop.com`. Building a dashboard from existing CSVs works offline.

## How to run

### Prebuilt app

1. Put your settings file (see [Configuration](#configuration)) in the same folder as the app.
2. **Windows:** double-click `CashCards LIVE.exe`. On first launch you may see "Windows protected your PC"; choose **More info → Run anyway**.
3. **macOS:**
   1. Unzip `CashCards LIVE.app.zip`.
   2. On first launch, right-click the app and choose **Open → Open**. You can also use **System Settings → Privacy & Security → Open Anyway**.
   3. The Mac build only runs on the chip type it was built on (Apple Silicon or Intel).
4. If the app doesn't find the settings file, click **Choose .env…** and select it. See the macOS note under [Troubleshooting](#troubleshooting).

### From source

Windows:

```
py "CCU LIVE Dashboard.py"
```

macOS / Linux:

```
python3 "CCU LIVE Dashboard.py"
```

On Windows you can also rename the file to `.pyw` and double-click it to run without a console window.

### Using the app

1. Click **Start**. If no settings file was found, you are asked to choose one.
2. Answer **Yes** to export fresh data, then enter how many days back to export (1–90). TikTok data usually lags 1–2 days. Answer **No** to skip straight to choosing existing CSVs.
3. On the first export, the app asks to connect TikTok Shop. Your browser opens the authorization page; sign in and click **Authorize**. The app waits up to 4 minutes for the callback. If the callback doesn't arrive, paste the code, or the whole URL of the page you landed on.
4. Choose one or more `live_minutes_*.csv` files. Files that are not minute files are skipped.
5. The dashboard is written to `dashboards/` and opened in your browser.

**Open folder** opens the app's folder.

## Configuration

All settings come from a plain-text settings file in `KEY=value` format, placed next to the app. The app does **not** read OS environment variables.

Blank lines and lines starting with `#` are ignored, and surrounding quotes are stripped from values.

### Settings file lookup

The app checks these names in order, in the app's folder:

```
.env, env.txt, env, config.env, tts.env, .env.txt, env.txt.txt, config.txt
```

If none of those exist, it uses the first file matching `*.env`, `env*`, `Env*`, `ENV*` or `config*` that is not a `.py`, `.pyw`, `.html`, `.csv` or `.json` file. You can also pick a file manually with **Choose .env…**.

If the chosen file is rich text (it starts with `{\rtf`, or is a `.rtf` / `.rtfd`), the app extracts the `TTS_*=value` pairs, writes them to `env.txt` in the same folder, and uses that file.

### Keys

| Key | Required | Set by | Purpose |
|---|---|---|---|
| `TTS_APP_KEY` | Yes | You | TikTok Shop Partner app key. |
| `TTS_APP_SECRET` | Yes | You | TikTok Shop Partner app secret. Used to sign API requests. |
| `TTS_SERVICE_ID` | No | You | Service ID for the authorization page. Defaults to the ID built into the code (`SERVICE_ID`). |
| `TTS_ACCESS_TOKEN` | No | App | Current access token. |
| `TTS_ACCESS_TOKEN_EXPIRES` | No | App | Access token expiry (Unix time). The token is refreshed when less than 1 hour remains. |
| `TTS_REFRESH_TOKEN` | No | App | Refresh token. If present, the app treats the login as already connected. |
| `TTS_REFRESH_TOKEN_EXPIRES` | No | App | Refresh token expiry (Unix time). Once it passes, you are asked to authorize again. |

Minimal example (placeholders only):

```
TTS_APP_KEY=your_app_key
TTS_APP_SECRET=your_app_secret
```

After the first sign-in, the app appends the four `TTS_*TOKEN*` lines under a comment header. It keeps everything else in the file as is.

> **Security:** anyone who has this file can pull the shop's data. Do not commit it, and share it only with people who should see the shop data.

### Settings in the code

These are constants near the top of `CCU LIVE Dashboard.py`, not user settings:

| Constant | Value | Meaning |
|---|---|---|
| `ACCOUNT_TYPES` | `OFFICIAL_ACCOUNTS`, `MARKETING_ACCOUNTS` | Account types exported |
| `WORKERS` | `6` | Sessions fetched in parallel |
| `CALLBACK_PORT` | `8765` | Local OAuth callback port |
| `SERVICE_ID` | built-in | Default authorization service ID |
| `currency` | `USD` | Currency requested from the API (argument to `run_export`) |

The dashboard has its own built-in account display names and host-detection rules, embedded in the compressed `DASHBOARD_B64` HTML. Hosts are detected from LIVE titles (for example "W/Name"), and a LIVE with two hosts counts for both. Changing these means editing the embedded HTML.

## Data and output files

Everything is created in the app's folder: the folder holding the `.py` / `.exe`, or the executable's own folder in a packaged build.

| Path | Contents |
|---|---|
| `exports/live_sessions_<start>_<end>.csv` | One row per LIVE session. Leading columns are `account, id, title, start_time, end_time`, followed by all API fields, flattened (nested keys joined with `_`, lists stored as JSON). |
| `exports/live_minutes_<start>_<end>.csv` | One row per minute. Leading columns are `account, live_id, live_title, minute_start, minute_end` (local time), followed by flattened per-minute metrics. This is the file the dashboard uses. |
| `exports/<name>_<HHMMSS>.csv` | Fallback name used when the normal CSV is locked by another program. |
| `exports/.cache/<live_id>.json` | Cached minute data for LIVEs that started more than 2 days ago. Never expires; delete it to force a re-fetch. |
| `dashboards/LIVE_dashboard_<YYYY-MM-DD_HHMM>.html` | Self-contained dashboard with the selected CSV data embedded. |
| `env.txt` | Written only when converting a rich-text settings file. |
| `tts_tokens.json` | Legacy token file. Written only if no settings file is loaded when tokens are saved. |

### Legacy token import

If the settings file has no `TTS_REFRESH_TOKEN`, the app looks for an older `tts_tokens.json` in these places:

- next to the settings file
- in the app folder
- in the app folder's parent
- in `~/Downloads`
- in `~/OneDrive/Downloads`

If it finds one, it copies the tokens into the settings file.

> **Note:** generated dashboards and CSVs contain shop sales data. Treat them as you would the settings file.

## Building the app

The build scripts are not in this repository; they come in a separate build kit. Both use PyInstaller with the same options, and you can reproduce them by hand.

The kit's scripts look for `CashCards LIVE Dashboard.pyw`, then the newest `*LIVE Dashboard*.pyw`, then the newest `*LIVE Dashboard*.py`, so `CCU LIVE Dashboard.py` is picked up.

### Windows

The kit script is `Build Windows EXE.bat`. To build by hand:

```
py -m pip install --upgrade pyinstaller truststore certifi
py -m PyInstaller --onefile --windowed --noconfirm --clean --name "CashCards LIVE" --hidden-import truststore --hidden-import certifi "CCU LIVE Dashboard.py"
```

The output is `dist/CashCards LIVE.exe`. The kit script writes it next to the script instead.

### macOS

The kit script is `Build Mac App.command`. It needs Python from python.org (with Tk) and builds in a temporary virtual environment. To build by hand:

```
python3 -m venv /tmp/ccl_build_venv
/tmp/ccl_build_venv/bin/python -m pip install --upgrade pip pyinstaller truststore certifi
/tmp/ccl_build_venv/bin/python -m PyInstaller --windowed --noconfirm --clean --name "CashCards LIVE" --hidden-import truststore --hidden-import certifi "CCU LIVE Dashboard.py"
xattr -cr "dist/CashCards LIVE.app"
```

Zip the `.app` before sharing it. The build runs only on Macs with the same chip type as the build machine.

Rebuild after any change to the source. To share the app, send the `.exe` or zipped `.app` together with a settings file; the recipient puts both in the same folder.

## Troubleshooting

### Certificate errors (`SSLCertVerificationError`)

The app starts with Python's default trust store, which includes the Windows store. If that store is empty, it tries these in order:

1. `certifi`, if installed
2. `/etc/ssl/cert.pem` (macOS)
3. `/etc/ssl/certs/ca-certificates.crt` (Debian/Ubuntu)
4. `/etc/pki/tls/certs/ca-bundle.crt` (Fedora/RHEL)

If verification still fails on a Mac, run **Install Certificates.command** in `/Applications/Python 3.x/` and try again.

### Other problems

| Symptom | Fix |
|---|---|
| "No settings file found next to this app" | Put `.env` or `env.txt` next to the app, or use **Choose .env…**. Finder won't create files named `.env`, so use `env.txt`. |
| "must contain TTS_APP_KEY=... and TTS_APP_SECRET=..." | Add both keys to the settings file. |
| "Couldn't read any settings from …" | The file is rich text with no readable keys. In TextEdit choose **Format → Make Plain Text** and save as `env.txt`. |
| "Port 8765 is busy" | Close other copies of the app and retry. |
| Browser approval never reaches the app | After 4 minutes the app asks you to paste the code or the full redirect URL. One way to get a code: in Partner Center, open your app's **Authorization** page, choose **Seller**, copy the authorization link, open it and approve. |
| "Saved login has expired" / "Couldn't refresh login" | Authorize again when prompted. The new tokens overwrite the old ones in the settings file. |
| "No shops are authorized for this login" | Authorize the app for the shop in TikTok Shop. |
| A session is "skipped … Usually still live or not processed yet" | The minute data isn't available yet. Export again later. |
| "None of the chosen files are live_minutes CSVs" | Choose `live_minutes_*.csv` files, not `live_sessions_*.csv`. |
| Windows "protected your PC" / macOS "can't be opened" | The app isn't code-signed. Use **More info → Run anyway** on Windows, or **right-click → Open** on macOS. |
| macOS build can't find the settings file, or files end up somewhere unexpected | In a packaged `.app`, the app folder resolves to the executable inside the bundle (`CashCards LIVE.app/Contents/MacOS`), not the folder containing the `.app`. Use **Choose .env…**, and use **Open folder** to see where `exports/` and `dashboards/` are written. |

Unexpected startup errors are shown in an error dialog, or printed to the console if Tk itself can't load.

## Project layout

```
.
├── CCU LIVE Dashboard.py     # The entire app: settings, OAuth, API client, export, dashboard builder, Tk GUI
├── CashCards LIVE.exe        # Prebuilt Windows app (PyInstaller --onefile)
├── CashCards LIVE.app.zip    # Prebuilt macOS app (zipped .app)
└── README.md
```

Main sections of `CCU LIVE Dashboard.py`:

- **config:** settings file discovery, RTF cleanup, `KEY=value` read and update.
- **http:** SSL context with CA fallback; JSON GET with retries.
- **tokens:** token get, refresh and storage; browser authorization via the local callback server.
- **signing / Client:** HMAC-SHA256 request signing; a client that retries on rate limits.
- **export:** session and per-minute fetch, cache, CSV writing.
- **dashboard:** `DASHBOARD_B64` (zlib-compressed, base64-encoded HTML) and `build_dashboard()`, which embeds the selected CSVs.
- **GUI:** Tkinter window, the Start flow and the log pane.

Files created at runtime (`exports/`, `dashboards/`, settings files, `tts_tokens.json`) contain credentials or shop data and should not be committed.
