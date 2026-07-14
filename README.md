# OpenObserve Windows Monitoring — Setup Report

**Host:** Windows machine — both the OpenObserve server and the OTel collector agent run on the same host, reachable at `http://localhost:5080`

**Goal:** Run a self-hosted [OpenObserve](https://openobserve.ai/) instance on Windows, ship Windows host metrics and Security Event Log data into it, build a dashboard, and alert on failed logins via Telegram.

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

OpenObserve OSS ships as a single self-contained binary — no separate database or dependencies to install. You can find their official documentation at [openobserve.ai/docs](https://openobserve.ai/docs/).

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

---

## 3. Windows Agent Installation

To get the tailored installation command for your specific OpenObserve instance:
1. In the OpenObserve UI, navigate to **Ingestion** (or **Data Sources** depending on your version) on the left-hand menu.
2. Select **Windows** from the list of available integrations.
3. The UI will display a pre-generated PowerShell script. This command is highly convenient because your `-URL` and `-AUTH_KEY` (Basic Auth) are already pre-populated for you.

Copy that command, open **PowerShell as Administrator**, and paste it to run:

```powershell
Invoke-WebRequest -Uri [https://raw.githubusercontent.com/openobserve/agents/main/windows/install.ps1](https://raw.githubusercontent.com/openobserve/agents/main/windows/install.ps1) -OutFile install.ps1
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

## 4. Viewing Logs in the UI

To explore your ingested Windows event data and host metrics natively:

1. Open the OpenObserve UI at `http://localhost:5080`.
2. On the left-hand navigation menu, click **Logs**.
3. In the stream selector dropdown (usually near the top left, labeled "Stream"), select your **`windows`** stream.
4. Directly below the stream selector, you can toggle the query bar between a **Visual Query Builder** (point-and-click filters) and **SQL Mode**. 
   * *Note: To extract nested JSON values using the `spath()` function you implemented on your dashboard, you must use SQL Mode.*

---

## 5. Sample Log Queries (SQL Mode)

Because your OpenTelemetry collector routes all four Windows event channels into a single `windows` stream, you will rely heavily on the `body_channel` and `body_event_id_id` fields to isolate signals, particularly for security monitoring. 

Paste any of the following queries directly into the SQL editor on the Logs page.

### Find Specific Failed Logins by Account Name
Use `spath()` to extract the raw JSON subject and look for failed logins (Event ID 4625) targeting a specific user.

```sql
SELECT
  _timestamp,
  body_channel,
  body_event_id_id,
  spath(body_details_subject, 'Account Name') as target_account,
  spath(body_details_subject, 'Logon Type') as logon_type
FROM "windows"
WHERE body_channel = 'Security'
  AND body_event_id_id = 4625
  AND spath(body_details_subject, 'Account Name') = 'adith'
ORDER BY _timestamp DESC
LIMIT 50
```

### Identify Potential Brute-Force Activity
To aggregate the noise and see which accounts are failing to log in most frequently over your selected time window (a standard metric for brute-force detection):

```sql
SELECT
  spath(body_details_subject, 'Account Name') as targeted_account,
  count(*) as failed_attempts
FROM "windows"
WHERE body_channel = 'Security'
  AND body_event_id_id = 4625
GROUP BY targeted_account
ORDER BY failed_attempts DESC
LIMIT 10
```

### Track Successful Logins
Useful for establishing a baseline of normal authentication activity.

```sql
SELECT
  _timestamp,
  spath(body_details_subject, 'Account Name') as logged_in_user,
  spath(body_details_subject, 'Account Domain') as domain,
  spath(body_details_subject, 'Logon ID') as logon_id
FROM "windows"
WHERE body_channel = 'Security'
  AND body_event_id_id = 4624
ORDER BY _timestamp DESC
LIMIT 50
```

### Search for Application Errors
If a Windows service crashes or an application throws a fault, it generally lands in the Application channel.

```sql
SELECT
  _timestamp,
  body_event_id_id,
  body_provider_name,
  body_level,
  body_message
FROM "windows"
WHERE body_channel = 'Application'
  AND body_level IN ('Error', 'Critical')
ORDER BY _timestamp DESC
LIMIT 50
```

### Full-Text Search Across All Logs
If you aren't sure which channel an event is in, or you just want to find a specific keyword (like a machine name or an IP address) across the entire stream:

```sql
SELECT *
FROM "windows"
WHERE match_all('LOQ-ENCELADUS')
ORDER BY _timestamp DESC
LIMIT 100
```
> **Note:** `match_all('keyword')` is OpenObserve's highly optimized full-text search function and is completely case-insensitive.

## 6. Dashboard: Import Custom JSON

The default community dashboard for Windows assumes that nested JSON fields (like `Account Name`) are auto-flattened into fields like `body_details_subject_account_name`. Depending on your OTel collector version, this might not happen, resulting in a `Search field not found` schema error.

To fix this, the dashboard JSON below has been customized to use `customQuery: true` and the `spath()` function to correctly extract the Account Name at query time.

### Import Instructions

1. Copy the JSON payload below and save it to a file named `Windows-Custom.dashboard.json` (or simply copy it to your clipboard).
2. In OpenObserve, navigate to **Dashboards** → **Import**.
3. Upload the file or paste the JSON directly.
4. Import it into your `default` organization.

### Custom Dashboard JSON

```json
{
  "version": 8,
  "dashboardId": "7482229072261545984",
  "title": "Windows",
  "description": "",
  "role": "",
  "owner": "",
  "created": "2026-07-13T00:27:03.912Z",
  "tabs": [
    {
      "tabId": "default",
      "name": "Default",
      "panels": [
        {
          "id": "Panel_ID7706110",
          "type": "area-stacked",
          "title": "Severity",
          "description": "",
          "config": {
            "show_legends": true,
            "legends_position": null,
            "base_map": {
              "type": "osm"
            },
            "map_view": {
              "zoom": 1,
              "lat": 0,
              "lng": 0
            }
          },
          "queryType": "sql",
          "queries": [
            {
              "query": "SELECT histogram(_timestamp) as \"x_axis_1\", severity as \"x_axis_2\", count(_timestamp) as \"y_axis_1\"  FROM \"windows\"  GROUP BY x_axis_1, x_axis_2 ORDER BY x_axis_1, x_axis_2",
              "vrlFunctionQuery": null,
              "customQuery": false,
              "fields": {
                "stream": "windows",
                "stream_type": "logs",
                "x": [
                  {
                    "label": " ",
                    "alias": "x_axis_1",
                    "type": "build",
                    "color": null,
                    "functionName": "histogram",
                    "args": [
                      {
                        "type": "field",
                        "value": {
                          "field": "_timestamp",
                          "streamAlias": null
                        }
                      }
                    ]
                  }
                ],
                "y": [
                  {
                    "label": " ",
                    "alias": "y_axis_1",
                    "type": "build",
                    "color": "#5960b2",
                    "functionName": "count",
                    "args": [
                      {
                        "type": "field",
                        "value": {
                          "field": "_timestamp",
                          "streamAlias": null
                        }
                      }
                    ]
                  }
                ],
                "z": [],
                "breakdown": [
                  {
                    "label": "Severity",
                    "alias": "x_axis_2",
                    "type": "build",
                    "color": null,
                    "args": [
                      {
                        "type": "field",
                        "value": {
                          "field": "severity",
                          "streamAlias": null
                        }
                      }
                    ]
                  }
                ],
                "filter": {
                  "filterType": "group",
                  "logicalOperator": "AND",
                  "conditions": []
                }
              },
              "config": {
                "promql_legend": "",
                "layer_type": "scatter",
                "weight_fixed": 1
              }
            }
          ],
          "layout": {
            "x": 0,
            "y": 0,
            "w": 64,
            "h": 16,
            "i": 1
          }
        },
        {
          "id": "Panel_ID6604910",
          "type": "area-stacked",
          "title": "Channel",
          "description": "",
          "config": {
            "show_legends": true,
            "legends_position": null,
            "base_map": {
              "type": "osm"
            },
            "map_view": {
              "zoom": 1,
              "lat": 0,
              "lng": 0
            }
          },
          "queryType": "sql",
          "queries": [
            {
              "query": "SELECT histogram(_timestamp) as \"x_axis_1\", body_channel as \"x_axis_2\", count(_timestamp) as \"y_axis_1\"  FROM \"windows\"  GROUP BY x_axis_1, x_axis_2 ORDER BY x_axis_1, x_axis_2",
              "vrlFunctionQuery": null,
              "customQuery": false,
              "fields": {
                "stream": "windows",
                "stream_type": "logs",
                "x": [
                  {
                    "label": " ",
                    "alias": "x_axis_1",
                    "type": "build",
                    "color": null,
                    "functionName": "histogram",
                    "args": [
                      {
                        "type": "field",
                        "value": {
                          "field": "_timestamp",
                          "streamAlias": null
                        }
                      }
                    ]
                  }
                ],
                "y": [
                  {
                    "label": " ",
                    "alias": "y_axis_1",
                    "type": "build",
                    "color": "#5960b2",
                    "functionName": "count",
                    "args": [
                      {
                        "type": "field",
                        "value": {
                          "field": "_timestamp",
                          "streamAlias": null
                        }
                      }
                    ]
                  }
                ],
                "z": [],
                "breakdown": [
                  {
                    "label": "Body Channel",
                    "alias": "x_axis_2",
                    "type": "build",
                    "color": null,
                    "args": [
                      {
                        "type": "field",
                        "value": {
                          "field": "body_channel",
                          "streamAlias": null
                        }
                      }
                    ]
                  }
                ],
                "filter": {
                  "filterType": "group",
                  "logicalOperator": "AND",
                  "conditions": []
                }
              },
              "config": {
                "promql_legend": "",
                "layer_type": "scatter",
                "weight_fixed": 1
              }
            }
          ],
          "layout": {
            "x": 64,
            "y": 0,
            "w": 64,
            "h": 16,
            "i": 2
          }
        },
        {
          "id": "Panel_ID5077210",
          "type": "area-stacked",
          "title": "Account name",
          "description": "",
          "config": {
            "show_legends": true,
            "legends_position": null,
            "base_map": {
              "type": "osm"
            },
            "map_view": {
              "zoom": 1,
              "lat": 0,
              "lng": 0
            }
          },
          "queryType": "sql",
          "queries": [
            {
              "query": "SELECT histogram(_timestamp) as \"x_axis_1\", count(_timestamp) as \"y_axis_1\", spath(body_details_subject, 'Account Name') as \"x_axis_2\" FROM \"windows\"  GROUP BY x_axis_1, x_axis_2",
              "vrlFunctionQuery": "",
              "customQuery": true,
              "fields": {
                "stream": "windows",
                "stream_type": "logs",
                "x": [
                  {
                    "label": " ",
                    "alias": "x_axis_1",
                    "column": "x_axis_1",
                    "type": "build",
                    "color": null,
                    "functionName": "histogram",
                    "args": [
                      {
                        "type": "field",
                        "value": {
                          "field": "_timestamp",
                          "streamAlias": null
                        }
                      }
                    ],
                    "treatAsNonTimestamp": false,
                    "showFieldAsJson": false
                  }
                ],
                "y": [
                  {
                    "label": " ",
                    "alias": "y_axis_1",
                    "column": "y_axis_1",
                    "type": "build",
                    "color": null,
                    "functionName": "count",
                    "args": [
                      {
                        "type": "field",
                        "value": {
                          "field": "_timestamp",
                          "streamAlias": null
                        }
                      }
                    ],
                    "treatAsNonTimestamp": false,
                    "showFieldAsJson": false
                  }
                ],
                "z": [],
                "breakdown": [
                  {
                    "label": "Body Details Subject Account Name",
                    "alias": "x_axis_2",
                    "column": "x_axis_2",
                    "type": "build",
                    "color": null,
                    "args": [
                      {
                        "type": "field",
                        "value": {
                          "field": "body_details_subject_account_name",
                          "streamAlias": null
                        }
                      }
                    ]
                  }
                ],
                "filter": {
                  "filterType": "group",
                  "logicalOperator": "AND",
                  "conditions": []
                }
              },
              "config": {
                "promql_legend": "",
                "layer_type": "scatter",
                "weight_fixed": 1
              },
              "joins": []
            }
          ],
          "layout": {
            "x": 128,
            "y": 0,
            "w": 64,
            "h": 16,
            "i": 3
          },
          "htmlContent": "",
          "markdownContent": "",
          "customChartContent": " // To know more about ECharts , \n// visit: [https://echarts.apache.org/examples/en/index.html](https://echarts.apache.org/examples/en/index.html) \n// Example: [https://echarts.apache.org/examples/en/editor.html?c=line-simple](https://echarts.apache.org/examples/en/editor.html?c=line-simple) \n// Define your ECharts 'option' here. \n// 'data' variable is available for use and contains the response data from the search result and it is an array.\noption = {  \n \n};\n  "
        },
        {
          "id": "Panel_ID4571710",
          "type": "table",
          "title": "Log",
          "description": "",
          "config": {
            "show_legends": true,
            "legends_position": null,
            "base_map": {
              "type": "osm"
            },
            "map_view": {
              "zoom": 1,
              "lat": 0,
              "lng": 0
            }
          },
          "queryType": "sql",
          "queries": [
            {
              "query": "SELECT histogram(_timestamp) as \"x_axis_1\", body_computer as \"x_axis_2\", body_channel as \"x_axis_3\", severity as \"x_axis_4\", spath(body_details_subject, 'Account Name') as \"x_axis_5\", body_message as \"x_axis_6\" FROM \"windows\"",
              "vrlFunctionQuery": "",
              "customQuery": true,
              "fields": {
                "stream": "windows",
                "stream_type": "logs",
                "x": [
                  {
                    "label": "Timestamp",
                    "alias": "x_axis_1",
                    "column": "x_axis_1",
                    "type": "build",
                    "color": null,
                    "functionName": "histogram",
                    "args": [
                      {
                        "type": "field",
                        "value": {
                          "field": "_timestamp",
                          "streamAlias": null
                        }
                      }
                    ],
                    "treatAsNonTimestamp": false,
                    "showFieldAsJson": false
                  },
                  {
                    "label": "Body Computer",
                    "alias": "x_axis_2",
                    "column": "x_axis_2",
                    "type": "build",
                    "color": null,
                    "args": [
                      {
                        "type": "field",
                        "value": {
                          "field": "body_computer",
                          "streamAlias": null
                        }
                      }
                    ],
                    "treatAsNonTimestamp": false,
                    "showFieldAsJson": false
                  },
                  {
                    "label": "Body Channel",
                    "alias": "x_axis_3",
                    "column": "x_axis_3",
                    "type": "build",
                    "color": null,
                    "args": [
                      {
                        "type": "field",
                        "value": {
                          "field": "body_channel",
                          "streamAlias": null
                        }
                      }
                    ],
                    "treatAsNonTimestamp": false,
                    "showFieldAsJson": false
                  },
                  {
                    "label": "Severity",
                    "alias": "x_axis_4",
                    "column": "x_axis_4",
                    "type": "build",
                    "color": null,
                    "args": [
                      {
                        "type": "field",
                        "value": {
                          "field": "severity",
                          "streamAlias": null
                        }
                      }
                    ],
                    "treatAsNonTimestamp": false,
                    "showFieldAsJson": false
                  },
                  {
                    "label": "Body Details Subject Account Name",
                    "alias": "x_axis_5",
                    "column": "x_axis_5",
                    "type": "build",
                    "color": null,
                    "args": [
                      {
                        "type": "field",
                        "value": {
                          "field": "body_details_subject_account_name",
                          "streamAlias": null
                        }
                      }
                    ],
                    "treatAsNonTimestamp": false,
                    "showFieldAsJson": false
                  },
                  {
                    "label": "Body Message",
                    "alias": "x_axis_6",
                    "column": "x_axis_6",
                    "type": "build",
                    "color": null,
                    "args": [
                      {
                        "type": "field",
                        "value": {
                          "field": "body_message",
                          "streamAlias": null
                        }
                      }
                    ],
                    "treatAsNonTimestamp": false,
                    "showFieldAsJson": false
                  }
                ],
                "y": [],
                "z": [],
                "breakdown": [],
                "filter": {
                  "filterType": "group",
                  "logicalOperator": "AND",
                  "conditions": []
                }
              },
              "config": {
                "promql_legend": "",
                "layer_type": "scatter",
                "weight_fixed": 1
              },
              "joins": []
            }
          ],
          "layout": {
            "x": 0,
            "y": 16,
            "w": 192,
            "h": 18,
            "i": 4
          },
          "htmlContent": "",
          "markdownContent": "",
          "customChartContent": " // To know more about ECharts , \n// visit: [https://echarts.apache.org/examples/en/index.html](https://echarts.apache.org/examples/en/index.html) \n// Example: [https://echarts.apache.org/examples/en/editor.html?c=line-simple](https://echarts.apache.org/examples/en/editor.html?c=line-simple) \n// Define your ECharts 'option' here. \n// 'data' variable is available for use and contains the response data from the search result and it is an array.\noption = {  \n \n};\n  "
        }
      ]
    }
  ],
  "variables": {
    "list": [],
    "showDynamicFilters": false
  }
}
```
---

## 7. Alert: Failed Windows Logins

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
| Destination | `telegram_alert` |

**Alternative considered:** switching **Alert Type** to **Real-time** fires instantly on every single matching event (no threshold, no period/frequency) instead of waiting for 3 in a 5-minute window — better for "notify me immediately," at the cost of more noise from occasional mistyped passwords.

---

## 8. Telegram Notification Destination

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

## 9. Known Issues / Notes

| Item | Status |
|---|---|
| `AUTH_KEY` (OpenObserve root Basic Auth) was shared in plaintext during setup | Rotate root password if not already done |
| Telegram bot token is a live secret | Keep out of shared chats/docs; regenerate via BotFather if exposed |
| Real-time vs. Scheduled alert trade-off for failed logins | Currently Scheduled (≥3 in 5 min); revisit if faster notification is wanted |
| `otel-collector` service does not auto-start after install/reinstall | Always run `Start-Service -Name otel-collector` manually afterward |
