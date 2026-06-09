# 🔋 SolisCloud Battery Monitor for n8n

An n8n workflow that monitors your Solis solar inverter battery and sends **Pushover push notifications** every time the battery drops through a 10% threshold while discharging.

## Features

- Polls SolisCloud API every 5 minutes
- Alerts at every 10% drop: 90%, 80%, 70%…
- Handles large drops — 95% → 65% fires alerts for 90%, 80%, and 70%
- **Quiet hours (12am–6am):** silences non-urgent alerts
- **Critical override:** battery ≤30% always alerts, even during quiet hours (Pushover emergency priority — buzzes until acknowledged)
- Supports multiple inverters on the same account

## Requirements

- [n8n](https://n8n.io) (self-hosted or cloud)
- [SolisCloud](https://www.soliscloud.com) account with API access enabled
- [Pushover](https://pushover.net) account

## Setup

### 1. Get SolisCloud API credentials
Log in to [soliscloud.com](https://www.soliscloud.com) → **Account → Basic Settings → API Management**

Copy your **KeyID** and **KeySecret**.

### 2. Get Pushover credentials
From your [Pushover dashboard](https://pushover.net):
- **User Key** — shown on your dashboard
- **Application Token** — create an application to get one

### 3. Import the workflow
In n8n: **Workflows → Import from file** → select `solis-battery-monitor.json`

### 4. Configure credentials

**First Code node** — replace these two lines:
```javascript
const keyId     = 'YOUR_KEY_ID';
const keySecret = 'YOUR_KEY_SECRET';
```

**Both Pushover HTTP Request nodes** — replace in the JSON query:
```
"token": "YOUR_APP_TOKEN"
"user":  "YOUR_USER_KEY"
```

### 5. Adjust settings (optional)

In the second Code node, at the top:

```javascript
const INTERVAL     = 10;   // alert every 10% — change to 5 for every 5%
const CRITICAL_SOC = 30;   // always alert at or below this %
```

For quiet hours (default 12am–6am):
```javascript
const isQuietHours = hour >= 0 && hour < 6;
```

For timezone (default UTC+8):
```javascript
const utc8 = new Date(now.getTime() + 8 * 60 * 60 * 1000);
//                                    ^ change 8 to your UTC offset
```

### 6. Test then activate
Use the **manual trigger** to test. Once working, enable the **Schedule Trigger** (every 5 minutes) and activate the workflow.

## Workflow Structure

```
[Schedule: 5min] ──┐
                   ├─→ [Code: Auth] → [HTTP: SolisCloud] → [Code: Thresholds] → [Switch]
[Manual Trigger] ──┘                                                                │
                                                               ┌────────────────────┤
                                                               │                    │
                                                    route=send │      route=critical│
                                                               ↓                    ↓
                                                    [Pushover Alert]   [Pushover Emergency]
                                                     priority: high      priority: emergency
```

## Notification Examples

**Normal alert:**
> 🔋 Mariden Resort
> Battery dropped below **80%**
> Current: **78%** — discharging at **1.2 kW**
> 🕐 2026-06-09 14:32:11 (UTC+8)

**Critical alert (any time, including quiet hours):**
> 🚨 Mariden Resort
> Battery dropped below **30%**
> Current: **27%** — discharging at **2.4 kW**
> 🕐 2026-06-10 03:15:44 (UTC+8)

## How SolisCloud Auth Works

SolisCloud requires a freshly computed HMAC-SHA1 signature on every request. Two non-obvious requirements not clearly documented:

- Use `application/json` (no charset) in the **signing string** even if the header includes charset
- Date must use **zero-padded day** — `09 Jun` not `9 Jun`

Signing string format:
```
POST\n{Content-MD5}\napplication/json\n{Date}\n{API path}
```

## License

MIT
