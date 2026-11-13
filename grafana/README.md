# Grafana

`deployment.yaml` runs Grafana with:

- `GF_SECURITY_ALLOW_EMBEDDING=true` so public dashboards can be iframed.
- `a5-images` ConfigMap mounted at `/usr/share/grafana/public/img/a5`, served at `/public/img/a5/<file>`. Used by the A5 Traffic dashboard's text panel. Regenerate from `traffic-display/app/static` with the command in the file's header.

## Dashboards

`dashboards/a5-traffic.json` is the A5 Traffic dashboard (v2 schema). Its `metadata.name` is `adpzd54`, the uid of the existing dashboard, so importing it **overwrites that dashboard in place** and the public URL keeps working.

Import: Dashboards → New → Import → paste the JSON → confirm the overwrite. After editing in the UI, export (Share → Export → JSON) and paste back here so the repo stays the source of truth.

Panels read from Redis (`redis-datasource`, uid `eflh9lnuvx8g0a`):

| Panel | Query |
|---|---|
| Current travel time, Delay, Status, Last checked | `HGETALL traffic_latest` |
| Travel time, Delay over free-flow, Worst delay today, Average by weekday | `XRANGE traffic_stats ${__from} ${__to}` |
| Recent alerts, "Traffic alerts" annotations | `XREVRANGE traffic_alerts` / `XRANGE traffic_alerts` |

Keys are written by [traffic-alerts](https://github.com/FredoAF/traffic-alerts).
