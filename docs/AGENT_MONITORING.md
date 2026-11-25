# Agent Monitoring

Agent Monitoring shows the servers that run the atglance agent: health, metrics, service checks,
requests and alerts. This page covers the monitoring controls (per server and per service),
restarting monitoring, alerts and their thresholds, notifications, how the data is stored and kept,
how to remove old servers and how to update agents from the console. Screens and APIs:
docs/console-agents.md. Protocol: docs/aacp-v1.md.

## Server health

| State | Meaning |
|---|---|
| Online | Connected, heartbeat on time, healthy |
| Degraded | Connected, but the agent reports degraded or a collector failed |
| Delayed | No heartbeat for `delayed_after` (90 s) |
| Offline | Connection closed, or no heartbeat for `offline_after` (180 s) |
| Paused | Monitoring turned off for this server (or the plugin is off): not offline, no alerts |
| Pending | Enrolled, never connected |
| Revoked | The agent's token was revoked |

## Monitoring per server

Servers page: the **Monitoring** switch on each row, the same switch in the row's **Settings**
panel, and **Monitoring on / off** for the selected rows (bulk). API:
`PUT /api/v1/agents/{id}/monitoring {"enabled": false}`. Allowed for the administrator, and for a
workspace admin on servers in workspaces they administer; other servers are "not found".

- **Off:** the agent gets `enabled: false` at once (or when it reconnects, in the welcome) and
  stays connected but sends nothing; its collectors idle. The console drops any report still on its
  way, the server shows **Paused**, its open alerts are resolved ("monitoring disabled") and no
  alert or notification opens for it. It is left out of availability, the dashboard counts ("N
  paused" is shown on its own) and outages. History is kept.
- **On:** the full config goes out with a new version; the server goes back through the normal
  states. The silence while paused never opens an offline alert.
- The plugin switch overrides everything: plugin off = every server paused.
- Each change is in the agent audit log (`agent.monitoring_disabled`, `agent.monitoring_enabled`)
  and the activity log.

## Monitoring per service

On a server's Settings panel (Services) and on the Service heartbeat page:

| State | Meaning |
|---|---|
| Monitored | Reported, alerts on (default for a watched service or a category switched on) |
| Not monitored | Not reported for this server, no alerts. History kept. |
| Monitored, alerts off | Shown, no alerts or notifications (noisy services) |

Precedence: **not monitored for this server > watch list > category switch**. The rule that decides
is shown next to each service. Agents 3.1.0 and newer stop reporting a service that is not monitored
(`collectors.services.ignore`); for older agents the console drops it. Turning monitoring or alerts
off resolves the service's open alerts. Each service can also have its own "down after" threshold,
missed-heartbeat intervals and a mute time.

A service that disappears from a server (uninstalled) is marked **not seen since …**, its alerts are
resolved, and it can be hidden. It is deleted after the history retention (`history_days`).

## Restarting monitoring

**Restart monitoring** (Request menu, or the Restart buttons in the Settings panel) restarts the
agent's collectors without restarting the agent: all of them, one (`system`, `disk`, `network`,
`services`, `apps`), or service monitoring for a category (the service lists are read again). It
shows on the Requests page. It needs bidirectional mode, a connected agent 3.1.0 or newer, and an
administrator or workspace admin; otherwise it is refused with the reason. **Restart agent** stays
the next step up. On an offline server run `sudo atglance agent monitoring restart` (or
`sudo atglance agent restart`) on the server.

The agent supervises its collectors: one that has not finished a run for 3 intervals (at least
90 s) is restarted by its watchdog and reported as event `collector.restarted`. Each collector's
state, last run and last error are on the Settings panel.

**Automatic recovery** (Site Settings > Agents, off by default): a server degraded by a failed
collector for the set number of minutes gets one Restart monitoring of that collector, at most once
an hour, recorded as event `collector.auto_restart`.

## Alerts and notifications

| Alert | Opens | Notification |
|---|---|---|
| Server offline | after the server has been silent for its offline after (per server, else Site Settings > Agents) | once per outage; "back online" includes how long it was down |
| Server degraded | at once | none |
| Service stopped or failed | when still down after "alert when a service is down for longer than" (default 60 s; 0 = at once; per service override) | once; "running again" includes how long it was down. Back within the threshold: an event only |
| Service heartbeat missed | not reported for `service_missed_intervals` x the service interval (per service override), only while the server is online | once |
| App check failing | after the check's failure threshold, in a row | once, and again when it recovers |

- Notifications go through the Notifications page groups (organization), name the server, its
  workspace, the service or check, the state, since when, the threshold and a link to the Servers
  page. Each kind can be switched off in Site Settings > Agents.
- Nothing is sent for a paused server, a server with alerts off or muted ("mute until", server
  Settings), or a service with alerts off or muted. Alerts and events are still recorded.
- **Repeat notifications** of open critical alerts every N minutes (Site Settings > Agents, 0 =
  never). **Acknowledge** on the Alerts page stops them and keeps the alert open; **Resolve**
  closes it (it opens again if the problem is still there).
- Flap suppression (120 s) holds back events and notifications, never the alert itself.
- Durations and thresholds use the console's clock (agent times more than 5 minutes ahead are
  ignored).

**Alerts page** (`/servers/alerts`): open alerts, or the last 7 days, filtered by server, kind and
severity, with Acknowledge and Resolve (administrator or workspace admin).

**App checks** (server Settings panel): add, edit and delete HTTP, TCP, TLS certificate and process
checks; saving sends them to the agent at once when it is connected.

## How the data is stored

| Where | What |
|---|---|
| TimescaleDB, `agent_metrics` | Raw metric samples (hypertable). |
| TimescaleDB, `agent_metrics_5m` | 5-minute averages of the raw samples (continuous aggregate). |
| TimescaleDB, `agent_check_results` | Results of service and app checks (hypertable). |
| TimescaleDB, `agent_heartbeats` | Heartbeat history (hypertable). |
| MySQL, latest tables | The newest value per server: `agent_heartbeats_latest`, `agent_metrics_latest`, `agent_service_status`, `agent_app_checks`. |
| MySQL, history tables | `agent_outages`, `agent_alerts`, `agent_sessions`, `agent_events`, `agent_commands`, `agent_command_output`. |
| MySQL, settings | `agent_service_settings` (per server and service: monitored, alerts, thresholds, mute, hidden). |

## Data retention

Super admin only: **Settings > Agents > Data retention**. Values are whole days.

| Setting | Applies to | Default | Min | Max |
|---|---|---|---|---|
| Raw metrics | `agent_metrics` | 7 | 1 | 90 |
| 5-minute averages | `agent_metrics_5m` | 90 | 2 | 730 |
| Heartbeat history | `agent_heartbeats` | 7 | 1 | 365 |
| Service and app check results | `agent_check_results` | 90 | 1 | 730 |
| Outages and alerts | `agent_outages`, `agent_alerts` | 90 | 7 | 730 |
| Events and connection sessions | `agent_events`, `agent_sessions` | 90 | 7 | 730 |
| Request (command) output | `agent_commands`, `agent_command_output` | 30 | 1 | 365 |

- 5-minute averages must be kept longer than raw metrics.
- Each setting shows the size of its data and the oldest record. If TimescaleDB cannot be reached the
  page says "TimescaleDB not reachable".
- **Save** stores the values and applies the TimescaleDB retention policies at once.
- **Clean up now** runs the same cleanup as the daily job and reports what it deleted. It is limited
  to 6 runs a minute.

Paused servers and services that are not monitored follow the same retention. Services gone
from their server ("not seen since") are deleted after `history_days`.

### The daily job

`agents:retention` runs every day at 03:40 (in the scheduler container). It applies the TimescaleDB
policies and deletes old MySQL history in batches of 1000:

- ended outages and resolved or closed alerts older than the outages and alerts setting;
- disconnected sessions and events older than the events setting;
- finished requests and their output older than the request output setting.

It never deletes:

- servers (agents);
- the latest tables and `agent_service_status`, `agent_app_checks`, `agent_groups`;
- open outages and open alerts.

The agent audit log follows the console's activity retention, not these settings.

## Removing old servers

Super admin only (`rbac_id` 100), on the **Servers** page, for the own organisation.

- The threshold is **Servers without a heartbeat** in Settings > Agents > Data retention (1 to 3650
  days, default 30). A server counts by its last heartbeat (else its last contact). Nothing is
  removed automatically.
- **Remove** shows only on an offline server whose last heartbeat is older than the threshold;
  the console refuses a younger one.
- **Remove old servers** removes all servers past the threshold. It shows how many it will remove
  and asks you to confirm.
- A connected server is never removed.

What happens to each server:

1. It is revoked first, so its token stops working at once.
2. Its rows are deleted from every MySQL agent table and from TimescaleDB (`agent_metrics`,
   `agent_check_results`, `agent_heartbeats`).
3. The server itself is deleted and the removal is written to the activity log.

Kept: the registered system and its configuration backups. If a TimescaleDB delete fails, the server
is still removed and its time series rows age out through the retention policies.

## Updating agents

On the **Servers** page, open a connected server's request panel (Details) and choose **Update agent
to <version>** in the Request list. The option is greyed out, with the reason, when the server
already runs the latest version or cannot be updated. There is no update button on the rows and
no "update all".

What is needed:

- The **CONFIGURE** permission, the same as for Requests.
- Bidirectional mode (Site Settings > Agents).
- The server is connected and its agent is 3.0.0 or newer.
- The latest release has a file with format `binary` for the server's OS and CPU, offered over https.

The console sends the agent a command with the file's URL and sha256. The agent downloads it, checks
the sha256, replaces itself and restarts. Progress is shown like a Request, and the server reconnects
with the new version. Only one update runs on a server at a time.

The release list comes from `ATGLANCE_CLI_RELEASES_URL` (default
`https://atglance.live/api/releases/cli/latest`), set in `composer/config/cli_releases.php`.

## Restarting an agent

On the Servers page, Request > **Restart agent** restarts the agent on a connected server. The
agent answers (the request shows as succeeded on the Requests page), then exits 3 seconds later;
the service manager starts it again and it reconnects within a minute. It needs bidirectional mode,
an administrator or workspace admin, and agent 3.0.0 or newer (the AACP command `agent.restart`);
older agents are refused with the version they run.

**Offline servers.** The console cannot reach an offline agent. Run `sudo atglance agent restart`
on the server (or `sudo atglance agent install` to repair the service and get a new agent token).
Agent 3.0.0 and newer clears a saved revoke on restart, so an agent stopped by a 403 tries its
token at once.

Servers not added from the console: restart the agent on the server (`sudo systemctl restart
atglance-agent`, or `Restart-Service AtGlanceAgent` on Windows). A revoked agent (3.0.0 and newer)
also tries its token again by itself every 15 minutes.

## Turning the plugin off and on

The Agent Monitoring switch on Admin Settings > Plugins also controls its containers, through the
service controller (`ce-atglance-controller`, the same sidecar that starts and stops MCP and Remote
Tunnel):

- **Off:** connected agents are told to stop reporting and their sessions close. Then
  `ce-atglance-agent-worker`, `ce-atglance-agent-gateway`, `ce-atglance-agent-monitoring` and
  `ce-atglance-tsdb` are stopped.
  Their data (the `tsdb-data` volume and the MySQL tables) is kept. `agents:health` (every minute)
  stops them again if an update (`docker compose up`) started them while the plugin is off.
- **On:** the four containers are started (TimescaleDB first, then the monitoring service and the
  gateway). Agents reconnect within a few minutes.
- **From the controller container**, the same switch without the web pages (for example while the
  console cannot be reached):

  ```
  docker exec ce-atglance-controller php artisan agents:plugin off
  docker exec ce-atglance-controller php artisan agents:plugin on
  docker exec ce-atglance-controller php artisan agents:plugin status   # switch and each container
  ```

  It is recorded in the activity log as "(command line)". The Plugins card is served by the console
  container, never by `agent-monitoring`, because turning it off stops that container.
- Only the switch decides. A licence problem never stops the containers.
- If the controller is not reachable, the switch is still saved and the page names the containers
  to start or stop with `docker compose`.
- A second console on the same Docker host sets `ATGLANCE_CONTAINER_PREFIX` on its controller
  (default `ce-atglance`) so it controls its own containers.

## The Agent Monitoring service

With `docker-compose.agents.yml`, the Servers pages run in their own container,
`ce-atglance-agent-monitoring`. That covers Servers, Service heartbeat, Requests and Alerts, their
poll endpoints, Settings > Agents, and the agents API `/api/v1/agents/*`. It uses the same image as
the console, with role `agent-monitoring`. Design: docs/design/AGENT_MONITORING_SERVICE.md.

- **Routing:** the console's Caddy sends `/servers*`, `/agents/*`, `/admin/agents/*` and
  `/api/v1/agents*` to it. Kong (`:8002`) passes `/api/v1/agents*` to Caddy. The agent gateway
  calls it for agent sessions.
- **Sign-in:** shared. It reads the console's `.env` (same `APP_KEY`) and the database sessions,
  so nobody signs in twice. Keep `SESSION_COOKIE` and the other `SESSION_*` values in `.env`, not
  only in a compose override, so both containers use them.
- **Jobs:** `agents:health` and `agents:retention` stay in the console scheduler. `agents:consume`
  runs in `agent-worker`.
- **When it is down:**
  - the Servers pages show **Monitoring service unavailable**, and the poll endpoints and the API
    answer `503 {"code": "MONITORING_UNAVAILABLE"}`;
  - the top bar shows "Servers (unavailable)";
  - the rest of the console keeps working;
  - agents stay connected to the gateway and their messages wait in Redis (`aacp:in`) until
    `agent-worker` catches up.
- **Health check:** `GET /up` inside the container. Restart policy `unless-stopped`.
- **Logs:** `docker logs ce-atglance-agent-monitoring` (`LOG_CHANNEL=stderr`).
- **Rollback:** set `ATGLANCE_AGENT_MONITORING_EMBEDDED=true` and
  `ATGLANCE_AGENT_MONITORING_UPSTREAM=127.0.0.1:8000` in `.env`, then `docker compose up -d`.
  The console serves the pages again. No data moves either way.
- Without `docker-compose.agents.yml` there is no separate service: the console serves Agent
  Monitoring itself.

## Navigation

**Servers** is in the top bar. Servers, Service heartbeat and Requests are no longer in the sidebar:
the Servers and Service heartbeat pages link to each other and to Requests.

## Troubleshooting

| Problem | Cause and fix |
|---|---|
| "Monitoring service unavailable" on Servers | `ce-atglance-agent-monitoring` is stopped or unhealthy: `docker compose ps`, `docker logs ce-atglance-agent-monitoring`, then `docker compose up -d`. Agent Monitoring switched off in Plugins also stops it. |
| "TimescaleDB not reachable" in Settings > Agents | The console cannot connect to TimescaleDB. Check the container and the TimescaleDB settings, then reload. Retention of MySQL history still works. |
| "The agent runs CLI x; remote update needs 3.0.0 or newer" | Update that agent on the server. |
| "Agents are in report-only mode" | Turn on bidirectional mode in Site Settings > Agents. |
| "The server is not connected; an update needs a live connection" | Wait until the agent reconnects. |
| "Could not read the latest release" | The release list cannot be read. The console retries after a minute. Check `ATGLANCE_CLI_RELEASES_URL`. |
| "The latest release has no file for <os> <arch>" | The release has no plain executable for that system. |
| "The release file is not offered over https, so the agent would refuse it" | The release file URL is http. Fix the release. |
| "The release file has no valid sha256" | The release entry is incomplete. Fix the release. |
| "An update is already running on this server" | Wait for the running update to finish (it expires after 10 minutes), then try again. |
