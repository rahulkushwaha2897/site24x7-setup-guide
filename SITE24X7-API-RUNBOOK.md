# Site24x7 API Runbook — Zoho OAuth

This runbook covers how to authenticate against the Site24x7 REST API using Zoho OAuth and perform common monitor management tasks (listing monitors, updating thresholds, managing groups, etc.).

---

## Prerequisites

- Access to [api-console.zoho.com](https://api-console.zoho.com) under the Zoho account linked to your Site24x7 MSP
- `curl` and `jq` installed locally
- Your BU's ZAAID (fetched in Step 4)

---

## Step 1 — One-Time OAuth App Setup

1. Go to [api-console.zoho.com](https://api-console.zoho.com)
2. Create a **Self Client** (if not already created)
3. Note your **Client ID** and **Client Secret**
4. Go to the **Generate Code** tab and use these scopes:

   ```
   Site24x7.admin.All,Site24x7.reports.All,Site24x7.bu.All,Site24x7.msp.All
   ```

5. Set expiry to **10 minutes** → click **Create** → copy the grant code

> The grant code is single-use and expires in 10 minutes. Complete Step 2 immediately after.

---

## Step 2 — Exchange Grant Code for Tokens (First Time Only)

```bash
curl -X POST "https://accounts.zoho.com/oauth/v2/token" \
  -d "grant_type=authorization_code" \
  -d "client_id=<YOUR_CLIENT_ID>" \
  -d "client_secret=<YOUR_CLIENT_SECRET>" \
  -d "redirect_uri=https://www.site24x7.com" \
  -d "code=<GRANT_CODE>"
```

Save both values from the response:
- **`access_token`** — valid for 1 hour
- **`refresh_token`** — permanent (store this securely)

---

## Step 3 — Refresh Access Token (Use This Every Session)

The access token expires in 1 hour. Use your refresh token to get a new one anytime:

```bash
curl -X POST "https://accounts.zoho.com/oauth/v2/token" \
  -d "grant_type=refresh_token" \
  -d "client_id=<YOUR_CLIENT_ID>" \
  -d "client_secret=<YOUR_CLIENT_SECRET>" \
  -d "refresh_token=<YOUR_REFRESH_TOKEN>"
```

Set the new token in your shell:
```bash
TOKEN="<NEW_ACCESS_TOKEN>"
```

---

## Step 4 — Find Your BU ZAAIDs

Each Business Unit (BU) in the MSP has a unique ZAAID. Fetch all BUs:

```bash
TOKEN="<YOUR_ACCESS_TOKEN>"

curl -s "https://www.site24x7.com/api/short/msp/customers" \
  -H "Authorization: Zoho-oauthtoken $TOKEN" \
  -H "Accept: application/json; version=2.0" | jq '.data[] | {name, zaaid}'
```

Pick the ZAAID for the BU you want (e.g. `PRTH-PRD`) and set it:

```bash
ZAAID="<YOUR_BU_ZAAID>"
```

---

## Step 5 — List All Monitors in a BU

```bash
curl -s "https://www.site24x7.com/api/monitors?zaaid=$ZAAID" \
  -H "Authorization: Zoho-oauthtoken $TOKEN" \
  -H "Accept: application/json; version=2.0" \
  -o /tmp/all_monitors.json
```

Filter by name prefix (e.g. all `prodcft-` servers):

```bash
jq '[.data[] | select(.display_name | test("^prodcft-")) | {name: .display_name, id: .monitor_id, type: .type}]' \
  /tmp/all_monitors.json
```

---

## Step 6 — Get Full Config of One Monitor

```bash
MONITOR_ID="<MONITOR_ID>"

curl -s "https://www.site24x7.com/api/monitors/${MONITOR_ID}?zaaid=$ZAAID" \
  -H "Authorization: Zoho-oauthtoken $TOKEN" \
  -H "Accept: application/json; version=2.0" | jq '.data'
```

---

## Step 7 — Update a Monitor (Threshold, Notifications, Groups, Tags)

```bash
curl -s -X PUT "https://www.site24x7.com/api/monitors/${MONITOR_ID}?zaaid=$ZAAID" \
  -H "Authorization: Zoho-oauthtoken $TOKEN" \
  -H "Accept: application/json; version=2.0" \
  -H "Content-Type: application/json" \
  -d '{
    "monitor_id": "<MONITOR_ID>",
    "type": "SERVER",
    "display_name": "<SERVER_NAME>",
    "threshold_profile_id": "<THRESHOLD_PROFILE_ID>",
    "notification_profile_id": "<NOTIFICATION_PROFILE_ID>",
    "setting_profile_id": "<SETTING_PROFILE_ID>",
    "user_group_ids": ["<USER_GROUP_ID>"],
    "monitor_groups": ["<GROUP_ID_1>", "<GROUP_ID_2>"],
    "tag_ids": ["<TAG_ID_1>", "<TAG_ID_2>"],
    "sm_poll_interval": 3,
    "log_needed": false,
    "perform_automation": false,
    "enable_uptime_monitoring": false
  }' | jq '.code, .message'
```

> Setting `monitor_groups` on the monitor payload automatically registers it in the group — no need to update the group object separately in most cases.

---

## Step 8 — List Monitor Groups

```bash
curl -s "https://www.site24x7.com/api/monitor_groups?zaaid=$ZAAID" \
  -H "Authorization: Zoho-oauthtoken $TOKEN" \
  -H "Accept: application/json; version=2.0" | \
  jq '[.data[] | select(.display_name | test("^CFTPay")) | {name: .display_name, id: .group_id}]'
```

---

## Step 9 — Add Monitors to a Group

**Fetch the current group config:**

```bash
GROUP_ID="<GROUP_ID>"

curl -s "https://www.site24x7.com/api/monitor_groups/${GROUP_ID}?zaaid=$ZAAID" \
  -H "Authorization: Zoho-oauthtoken $TOKEN" \
  -H "Accept: application/json; version=2.0" | jq '.data' > /tmp/group.json
```

**Edit `/tmp/group.json`** to add your monitor IDs to the `monitors` array, then PUT it back:

```bash
curl -s -X PUT "https://www.site24x7.com/api/monitor_groups/${GROUP_ID}?zaaid=$ZAAID" \
  -H "Authorization: Zoho-oauthtoken $TOKEN" \
  -H "Accept: application/json; version=2.0" \
  -H "Content-Type: application/json" \
  -d @/tmp/group.json | jq '.code, .message'
```

---

## Key Reference

| Item | Detail |
|---|---|
| **Auth header** | `Authorization: Zoho-oauthtoken <access_token>` |
| **BU scoping** | All API calls require `?zaaid=<BU_ZAAID>` as a query parameter |
| **Access token TTL** | 1 hour — use Step 3 to refresh |
| **Refresh token TTL** | Permanent — store securely |
| **Base URL** | `https://www.site24x7.com/api/` |
| **Accept header** | `Accept: application/json; version=2.0` |

---

## Quick Session Start (Copy-Paste)

```bash
# 1. Refresh token
curl -s -X POST "https://accounts.zoho.com/oauth/v2/token" \
  -d "grant_type=refresh_token" \
  -d "client_id=<CLIENT_ID>" \
  -d "client_secret=<CLIENT_SECRET>" \
  -d "refresh_token=<REFRESH_TOKEN>" | jq '.access_token'

# 2. Set vars
TOKEN="<paste access_token here>"
ZAAID="<your BU ZAAID>"

# 3. Start working
curl -s "https://www.site24x7.com/api/monitors?zaaid=$ZAAID" \
  -H "Authorization: Zoho-oauthtoken $TOKEN" \
  -H "Accept: application/json; version=2.0" | jq '[.data[] | {name: .display_name, id: .monitor_id}]'
```
