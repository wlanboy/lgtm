# Troubleshooting: Loki (Log-Aggregation)

Incident-Runbook für Grafana Loki `3.7.1` in diesem Stack. Ergänzt die Kurzsektion in [readme.md](readme.md#troubleshooting) um eine tiefergehende Analyse für Produktionsprobleme.

Relevante Endpunkte in diesem Setup:

| Zweck | URL/Port |
|---|---|
| Loki HTTP API | http://localhost:3100 |
| Ready-Check | http://localhost:3100/ready |
| Interne Metriken | http://localhost:3100/metrics |
| Push-API (von Alloy genutzt) | http://loki:3100/loki/api/v1/push |

Konfiguration: `config/loki-config.yaml` (via `./config:/mnt/config` gemountet). Log-Quelle: Alloy liest `/var/log/*log` vom Host (siehe [shared/config.alloy](shared/config.alloy)).

---

## 1. Sofort-Diagnose (erste 5 Minuten)

- [ ] `curl -s http://localhost:3100/ready` → muss `ready` liefern
- [ ] `docker compose ps loki alloy` → beide Container `healthy`/`running`?
- [ ] `docker compose logs --tail=200 loki` → auf `level=error`, `panic`, `too many` scannen
- [ ] Grafana → Explore → Loki → `{job="varlogs"}` → kommen aktuelle Log-Zeilen an?
- [ ] Alloy Pipeline-Graph (http://localhost:12345) → Komponente für den Loki-Writer prüfen

---

## 2. Incident-Checkliste: "Logs fehlen / kommen nicht an" (Write-Pfad)

1. **Ist Loki erreichbar?**
   ```bash
   curl -v http://localhost:3100/ready
   ```
   Bei Verbindungsfehler: Container-Status, Netzwerk zwischen Alloy und Loki (`alloy → loki:3100`), Portmapping prüfen.

2. **Wird der Push abgelehnt (HTTP-Statuscode prüfen)?**
   - **429 Too Many Requests** → Rate-Limit greift. Relevante Limits in `loki-config.yaml`: `ingestion_rate_mb`, `ingestion_burst_size_mb`, `per_stream_rate_limit`. Vor dem Erhöhen prüfen, *warum* das Volumen so hoch ist (z. B. Log-Loop, Debug-Logging vergessen auszuschalten).
   - **400 Bad Request "entry too far behind" / out of order** → Zeitstempel-Validierung schlägt fehl. Ursache meist **Uhrzeit-Drift (Clock Skew)** zwischen Log-Quelle und Loki-Host. NTP-Sync auf allen Hosts prüfen. Mit Standardeinstellung (`unordered_writes: true`) werden nur Einträge älter als die Hälfte von `max_chunk_age` relativ zum neuesten Eintrag im Stream abgelehnt.
   - **500 Internal Server Error** → Ingester-seitiges Problem, siehe Punkt 4.

3. **Ist die maximale Anzahl aktiver Streams erreicht?**
   - Symptom: `max streams limit exceeded` in den Logs.
   - Aktive Streams liegen komplett im Arbeitsspeicher der Ingester → zu viele Streams = OOM-Risiko.
   - **Wichtig:** Nicht einfach `max_global_streams_per_user` hochsetzen, um hohe Cardinality "wegzukonfigurieren" — das führt zu extrem vielen kleinen Chunks und massiv schlechterer Query-Performance. Stattdessen die Label-Struktur der Quelle fixen (siehe Punkt 5).
   - Faustregel: Selbst bei mehreren hundert TB/Tag sollten global nicht mehr als ~300.000 aktive Streams pro Tenant nötig sein.

4. **Timeouts beim Schreiben?**
   - Client-seitigen Timeout erhöhen (Alloy: `remote_write` Timeout-Konfiguration)
   - Ressourcenauslastung des Loki-Containers prüfen: `docker stats loki` — CPU-Starvation führt dazu, dass Loki nicht mehr rechtzeitig antwortet
   - Netzwerklatenz zwischen Alloy/App und Loki prüfen

5. **Zu hohe Label-Cardinality?**
   - Prüfen, ob dynamische Werte (User-IDs, Request-IDs, Timestamps, IPs) versehentlich als **Label** statt als Log-Inhalt verwendet werden — das ist der häufigste Grund für Stream-Explosion in Loki.
   - In `shared/config.alloy` die `local.file_match`-Labels auf statische, niedrig-kardinale Werte beschränken (z. B. `job`, `container`, nicht `request_id`).

---

## 3. Incident-Checkliste: "Query ist langsam / schlägt fehl" (Read-Pfad)

- [ ] Zeitraum der LogQL-Query eingrenzen — große Zeitfenster über viele Streams sind die häufigste Ursache für langsame/fehlschlagende Queries
- [ ] Regex-/Filter-Ausdrücke prüfen (`|~ "regex"` ist teurer als `|= "substring"`)
- [ ] `docker compose logs loki | grep -i "query"` auf Timeout- oder "too many outstanding requests"-Meldungen prüfen
- [ ] Bei wiederkehrenden Dashboards: teure Queries durch gezieltere Label-Filter oder kürzere Zeitfenster ersetzen

---

## 4. Weitere bekannte Fehlerbilder

| Symptom | Wahrscheinliche Ursache | Maßnahme |
|---|---|---|
| Loki-Container bleibt "unhealthy" / startet nicht | Permission-Problem auf `loki-data`-Volume | `docker compose logs loki`; als letzter Schritt `docker compose down -v && docker compose up -d` (⚠️ löscht gespeicherte Logs) |
| Grafana Explore zeigt "No data" bei existierenden Logs | Falscher Label-Filter oder Zeitraum | Mit breitem Filter starten (`{job="varlogs"}`) und schrittweise eingrenzen |
| Alloy sammelt keine Docker-Container-Logs | Docker-Socket nicht erreichbar | `docker compose exec alloy ls -la /var/run/docker.sock`; Alloy neu starten |
| Plötzlicher Anstieg der Ingester-Speichernutzung | Stream-Cardinality-Explosion (siehe Punkt 5 oben) | Betroffene Labels identifizieren, Quelle korrigieren, ggf. Ingester neu starten |
| Logs erscheinen stark verzögert | Chunk-Flush-Intervall / Backpressure durch Rate-Limiting | `ingestion_rate_mb` und Ingester-Ressourcen prüfen |

---

## 5. Nützliche Kommandos & LogQL-Snippets

```bash
# Health- und Statusendpunkte
curl -s http://localhost:3100/ready
curl -s http://localhost:3100/metrics | grep -E "loki_ingester_streams|loki_discarded_samples_total"

# Live-Logs verfolgen
docker compose logs -f loki

# Volume-Zustand prüfen
docker volume inspect lgtm_loki-data
```

Nützliche LogQL-Abfragen in Grafana Explore:

```logql
# Fehler in System-Logs
{job="varlogs"} |= "error"

# Log-Volumen pro Job über Zeit (zur Cardinality-/Volumen-Analyse)
sum by (job) (count_over_time({job=~".+"}[5m]))

# Container-Logs eines bestimmten Services
{container="alloy"} |= "error"
```

Wichtige interne Metriken (über Prometheus-Datasource, Job `loki`):

```promql
rate(loki_discarded_samples_total[5m])   # verworfene Log-Zeilen nach Grund (Label: reason)
loki_ingester_memory_streams             # aktuell aktive Streams im Speicher
```

---

## 6. Eskalationskriterien

- Anhaltender Datenverlust durch `loki_discarded_samples_total` trotz korrigierter Limits
- Wiederkehrende OOM-Kills durch Stream-Cardinality trotz Label-Bereinigung
- Query-Latenzen, die SLOs verletzen, trotz eingegrenzter Zeiträume und Filter

---

## Links

- [Manage and debug errors (Loki Doku)](https://grafana.com/docs/loki/latest/operations/troubleshooting/)
- [Troubleshoot log ingestion (WRITE)](https://grafana.com/docs/loki/latest/operations/troubleshooting/troubleshoot-ingest/)
- [Troubleshoot log queries (READ)](https://grafana.com/docs/loki/latest/query/troubleshoot-query/)
- [Troubleshoot Loki operations](https://grafana.com/docs/loki/latest/operations/troubleshooting/troubleshoot-operations/)
- [Loki Docker Setup](https://grafana.com/docs/loki/latest/setup/install/docker/)
