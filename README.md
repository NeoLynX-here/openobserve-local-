# OpenObserve Windows Monitoring — Setup Report

**Host:** Windows machine — both the OpenObserve server and the OTel collector agent run on the same host, reachable at `http://localhost:5080`

**Goal:** Run a self-hosted OpenObserve instance on Windows, ship Windows host metrics and Security Event Log data into it, build a dashboard, and alert on failed logins via Telegram.

---

## 1. Architecture Overview

OpenObserve OSS runs locally as a standalone Windows binary. An OpenTelemetry Collector (`otelcol-contrib` v0.90.1), installed as a separate Windows Service via OpenObserve's official agent script, scrapes host metrics and tails four Windows Event Log channels, then ships everything to OpenObserve's `default` org under a single stream named `windows`.

| Signal | Receiver(s) | Destination stream |
|---|---|---|
| CPU, memory, disk, network, etc. | `hostmetrics` | `windows` (metrics) |
| Memory/CPU perf counters | `windowsperfcounters/memory`, `windowsperfcounters/processor` | `windows` (metrics) |
| Application, Security, Setup, System event logs | `windowseventlog/application`, `windowseventlog/security`, `windowseventlog/setup`, `windowseventlog/system` | `windows` (logs) |

---

## 2. OpenObserve Server Installation (Windows)

OpenObserve OSS ships as a single self-contained binary — no separate database or dependencies to install.

```powershell
# Download the OSS Windows binary (check https://openobserve.ai/downloads for the current version — v0.91.1 at time of writing)
Invoke-WebRequest -Uri "https://github.com/openobserve/openobserve/releases/download/v0.91.1/openobserve-v0.91.1-windows-amd64.zip" -OutFile "openobserve.zip"
Expand-Archive -Path "openobserve.zip" -DestinationPath .

# Set root credentials — required on first run only
$env:ZO_ROOT_USER_EMAIL = "root@example.com"
$env:ZO_ROOT_USER_PASSWORD = "<your-password>"
.\openobserve.exe
```

Once running, the UI is at `http://localhost:5080` — log in with the root email/password set above.

> This section documents OpenObserve's standard published install method for Windows. If the actual commands used differed (e.g. Docker Desktop instead of the raw binary), swap this block for the real ones.

## 3. Windows Agent Installation

```powershell
Invoke-WebRequest -Uri https://raw.githubusercontent.com/openobserve/agents/main/windows/install.ps1 -OutFile install.ps1
.\install.ps1 -URL http://localhost:5080/api/default/ -AUTH_KEY <REDACTED_BASIC_AUTH_KEY>
```

The script downloads `otelcol-contrib`, writes a config to `C:\otel-collector\otel-config.yaml`, and registers/starts a Windows Service named `otel-collector` running:

```
otelcol-contrib.exe --config=C:\otel-collector\otel-config.yaml
```

The exporter pushes via `otlphttp` to the `-URL` above, tagging everything with `stream-name: windows` and `Authorization: Basic <AUTH_KEY>`.

> **Note:** `AUTH_KEY` is a Base64-encoded Basic Auth credential for the OpenObserve root account. Treat it as a real password — rotate it from OpenObserve's UI (Settings → Users) if it was ever shared outside a secure channel.

The script registers the `otel-collector` Windows Service but does not start it automatically — a manual start is required after install (and after any reinstall):

```powershell
Start-Service -Name otel-collector
Get-Service otel-collector   # confirm Status: Running
```

---

## 4. Dashboard: Import and Field-Mapping Fix

### Import

Used OpenObserve's community dashboard repository, which has a dashboard built specifically for this Windows OTel agent:

1. Dashboard JSON: `https://raw.githubusercontent.com/openobserve/dashboards/main/Windows/Windows.dashboard.json`
2. OpenObserve → **Dashboards → Import** → pasted the URL (or uploaded the downloaded file) → imported into the `default` org.

### Issue: broken field reference

Two panels ("Account name" and the "Log" table) errored with:
```
Search field not found: Schema error: No field named body_details_subject_account_name.
```

**Root cause:** `body_details_subject` is ingested as a single field whose *value* is a raw JSON string (e.g. `{"Account Domain":"LOQ-ENCELADUS","Account Name":"adith","Logon ID":"0x721E2",...}`). Because the inner JSON keys contain spaces and capitals ("Account Name"), OpenObserve does not auto-flatten them into separate indexed fields the way it does for clean snake_case JSON — so `body_details_subject_account_name` never existed as a real field. The imported dashboard was built assuming a different ingestion-time flattening than what this collector version actually produces.

**Fix:** extract the value at query time with OpenObserve's `spath()` function, applied directly in each panel's SQL query (with `customQuery: true`):

```sql
SELECT
  histogram(_timestamp) as "x_axis_1",
  count(_timestamp) as "y_axis_1",
  spath(body_details_subject, 'Account Name') as "x_axis_2"
FROM "windows"
GROUP BY x_axis_1, x_axis_2
```

Applied the same pattern to the "Log" table panel's `x_axis_5` column. Both panels render correctly now.

> **Note:** the dashboard JSON's `fields.breakdown`/`fields.x` metadata (which drives the visual **Build** mode UI) still references the old dead field name for these two panels. This is harmless while `customQuery: true` is set — but toggling either field from **Raw/SQL** back to **Build** mode in the UI would regenerate a query from that stale metadata and reintroduce the error. Leave those two fields alone.

---

## 5. Alert: Failed Windows Logins

Filters on Windows Security Event ID **4625** ("An account failed to log on") rather than text-matching "fail", since all four event log channels share the one `windows` stream.

| Setting | Value |
|---|---|
| Name | `Failed_Windows_Logins` |
| Stream Type / Name | Logs / `windows` |
| Alert Type | Scheduled |
| Conditions | `body_channel = 'Security'` AND `body_event_id_id = 4625` |
| Alert if No. of events | `>= 3` |
| Period (look-back window) | 5 minutes |
| Frequency | 1 minute |
| Destination | `telegram_alert` (see §5) |

**Alternative considered:** switching **Alert Type** to **Real-time** fires instantly on every single matching event (no threshold, no period/frequency) instead of waiting for 3 in a 5-minute window — better for "notify me immediately," at the cost of more noise from occasional mistyped passwords.

---

## 6. Telegram Notification Destination

### Bot setup
1. Created a bot via **@BotFather** in Telegram → obtained a bot token.
2. Messaged the bot directly, then retrieved the chat ID:
   ```
   curl "https://api.telegram.org/bot<TOKEN>/getUpdates"
   ```
   Chat ID obtained: `983654387`.
3. Verified delivery before wiring up OpenObserve:
   ```
   curl "https://api.telegram.org/bot<TOKEN>/sendMessage?chat_id=983654387&text=test"
   ```

### Template
**Management → Templates → Add Template**, name `telegram_template`, type **Webhook**:
```json
{
  "chat_id": "983654387",
  "text": "🚨 {alert_name} triggered\nStream: {stream_name}\nCondition: {alert_operator} {alert_threshold}\nCount: {alert_count}\nTime: {alert_start_time}"
}
```

### Destination
**Management → Alert Destinations → Add Destination**, name `telegram_alert`, type **Webhook**:
- URL: `https://api.telegram.org/bot<TOKEN>/sendMessage`
- Method: `POST`
- Template: `telegram_template`

Attached to the `Failed_Windows_Logins` alert's **Destination** field.

---

## 7. Known Issues / Notes

| Item | Status |
|---|---|
| `AUTH_KEY` (OpenObserve root Basic Auth) was shared in plaintext during setup | Rotate root password if not already done |
| Telegram bot token is a live secret | Keep out of shared chats/docs; regenerate via BotFather if exposed |
| Two dashboard panels have stale `fields` metadata under the hood | Don't toggle those specific fields to Build mode in the UI |
| Real-time vs. Scheduled alert trade-off for failed logins | Currently Scheduled (≥3 in 5 min); revisit if faster notification is wanted |
| `otel-collector` service does not auto-start after install/reinstall | Always run `Start-Service -Name otel-collector` manually afterward |
