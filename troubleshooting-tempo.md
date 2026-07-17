# Troubleshooting: Tempo (Distributed Tracing)

Incident-Runbook für Grafana Tempo `2.10.4` in diesem Stack. Ergänzt die Kurzsektion in [readme.md](readme.md#troubleshooting) um eine tiefergehende Analyse für Produktionsprobleme.

Relevante Endpunkte in diesem Setup:

| Zweck | URL/Port |
|---|---|
| Tempo HTTP API / Ready | http://localhost:3200 |
| Tempo gRPC | localhost:9095 |
| OTLP gRPC Ingest | localhost:4317 |
| OTLP HTTP Ingest | localhost:4318 |
| Jaeger Ingest | localhost:14268 |
| Zipkin Ingest | localhost:9411 |
| Alloy Pipeline UI | http://localhost:12345 |
| Memcached (Search-Frontend-Cache) | localhost:11211 |

Konfiguration: [config/tempo.yaml](config/tempo.yaml) — Beachte: `block_retention: 1h` und `max_block_duration: 5m` sind bewusst niedrig für den Demo-/Dev-Betrieb gesetzt. In Produktion deutlich höher konfigurieren.

---

## 1. Sofort-Diagnose (erste 5 Minuten)

- [ ] `curl -s http://localhost:3200/ready` → muss `ready` liefern
- [ ] `curl -s http://localhost:3200/status/endpoints` → prüft aktive Endpunkte
- [ ] `docker compose ps tempo memcached alloy` → alle Container `healthy`/`running`?
- [ ] `docker compose logs --tail=200 tempo` → auf `error`, `panic`, `refused`, `OOM` scannen
- [ ] Alloy Pipeline-Graph unter http://localhost:12345 öffnen → rote/fehlerhafte Komponenten?
- [ ] Grafana → Explore → Tempo → Search → liefert ein Test-Trace einen Treffer?

---

## 2. Incident-Checkliste: "Keine/fehlende Traces"

Reihenfolge von der Quelle bis zur Anzeige durchgehen (Instrumentierte App → Alloy → Tempo → Grafana):

1. **Erzeugt die Anwendung überhaupt Spans?**
   - Sampling-Konfiguration in der App/SDK prüfen (z. B. `parentbased_traceidratio`)
   - Bei `k6-tracing`: läuft der Container? `docker compose logs k6-tracing`

2. **Kommen Spans bei Alloy an?**
   - Alloy UI (http://localhost:12345) → Komponente für OTLP-Receiver prüfen
   - Alloy-Metriken abfragen (über Prometheus, Datasource "prometheus"):
     ```promql
     rate(receiver_refused_spans_ratio_total[5m])
     rate(exporter_send_failed_spans_ratio_total[5m])
     ```
     Beide sollten `0` sein. Werte > 0 bedeuten: Alloy verwirft Spans oder kann sie nicht an Tempo weiterleiten.

3. **Kommen Spans bei Tempo an?**
   - Tempo-Metrik prüfen:
     ```promql
     rate(tempo_receiver_refused_spans[5m])
     rate(tempo_discarded_spans_total[5m])
     ```
     `tempo_receiver_refused_spans > 0` → Rate-Limiting oder Overrides greifen (`max_bytes_per_trace`, `max_traces_per_user` in `overrides:`).
   - Distributor-Logs auf `pusher failed to consume trace data` prüfen → deutet auf überschrittenes Trace-Size-Limit hin.

4. **Falsches Protokoll/Port?**
   - OTLP gRPC (`4317`) vs. OTLP HTTP (`4318`) nicht verwechseln — häufigste Fehlerquelle bei neu instrumentierten Apps.
   - Container-interne Anwendungen müssen an `tempo:4317`/`alloy:4317` senden (Service-Name im Docker-Netzwerk), nicht `localhost`.

5. **Sind die Traces schon wieder abgelaufen?**
   - `block_retention: 1h` in diesem Stack → alte Traces sind nach einer Stunde weg. Bei "Trace war gestern da, heute nicht mehr" ist das meist die Ursache, kein Bug.

6. **Grafana-Datasource-Problem?**
   - Tempo-Datasource-URL muss auf den **HTTP-Port 3200** zeigen, nicht auf gRPC (9095) — Grafana leitet die gRPC-Verbindung intern vom HTTP-Endpunkt ab.
   - Bei TLS-Fehlern: Zertifikat-Mismatch zwischen Grafana und Tempo prüfen (in diesem Stack standardmäßig kein TLS, also nur relevant bei eigenen Anpassungen).

---

## 3. Weitere bekannte Fehlerbilder

| Symptom | Wahrscheinliche Ursache | Maßnahme |
|---|---|---|
| Tempo-Container startet nicht / crasht sofort | Volume-Permission auf `tempo-data` (Tempo läuft als User `10001`) | Prüfen ob `init`-Container erfolgreich lief: `docker compose logs init`. Ggf. `docker compose up -d init` erneut ausführen |
| `error using pageFinder` bei Suche (HTTP 500) | Korrupter oder inkonsistenter Block im lokalen Storage | Tempo-Logs auf Compactor-Fehler prüfen; betroffenen Block ggf. aus `tempo-data`-Volume entfernen (Datenverlust für diesen Block) |
| `too many jobs in the queue` / HTTP 429 bei Suche | Query-Frontend-Kapazität erschöpft (viele parallele/teure Suchen) | Zeitraum der Suche eingrenzen, `duration_slo`/`throughput_bytes_slo` in `query_frontend` prüfen, Suchlast reduzieren |
| Service-Graph/Span-Metrics fehlen in Grafana | Metrics-Generator schreibt nicht erfolgreich zu Prometheus | `docker compose logs tempo \| grep -i "remote write"`; prüfen ob Prometheus mit `--web.enable-remote-write-receiver` läuft (ist in diesem Stack Standard); ~2 Minuten Vorlaufzeit nach Start abwarten |
| Trace-Suche zeigt inkonsistente Ergebnisse bei langlaufenden Traces | Trace erstreckt sich über mehrere Blöcke (durch `max_block_duration: 5m`) | Erwartetes Verhalten bei kurzem `max_block_duration`; für Produktion höher konfigurieren |
| Hoher Speicherverbrauch / OOM bei Tempo | Sehr große einzelne Spans/Attribute oder zu viele parallele Traces im Head-Block | `max_attribute_bytes` / Span-Size-Limits in den `overrides` setzen; Ressourcenlimits im Compose-File erhöhen |
| Grafana zeigt "no data" nur bei Service Graph | Zu kurze Wartezeit nach Deploy | Service Graphs brauchen laut Doku ca. 2 Minuten Anlaufzeit, bevor Daten erscheinen |

---

## 4. Nützliche Kommandos

```bash
# Health- und Statusendpunkte
curl -s http://localhost:3200/ready
curl -s http://localhost:3200/status/endpoints
curl -s http://localhost:3200/metrics | grep tempo_receiver_refused_spans

# Live-Logs verfolgen
docker compose logs -f tempo
docker compose logs -f alloy

# In den Tempo-Container schauen
docker compose exec tempo sh
ls -la /var/tempo/blocks     # gespeicherte Blöcke
ls -la /var/tempo/wal        # Write-Ahead-Log (noch nicht kompaktierte Traces)

# Volume-Zustand prüfen
docker volume inspect lgtm_tempo-data

# Memcached erreichbar? (Search-Frontend-Cache)
docker compose exec tempo sh -c "nc -zv memcached 11211"
```

---

## 5. Eskalationskriterien

- Datenverlust durch korrupte Blöcke oder versehentliches `docker compose down -v`
- Anhaltende `tempo_receiver_refused_spans` trotz angepasster Limits → Kapazitätsproblem, Skalierung prüfen
- OOM-Kills des Tempo-Containers unter Last → Ressourcenlimits und Ingestion-Rate gemeinsam mit Team abstimmen

---

## Links

- [Troubleshoot Tempo (offizielle Doku)](https://grafana.com/docs/tempo/latest/troubleshooting/)
- [Unable to find traces](https://grafana.com/docs/tempo/latest/troubleshooting/querying/unable-to-see-trace/)
- [Troubleshoot Tempo data source in Grafana](https://grafana.com/docs/grafana/latest/datasources/tempo/troubleshooting/)
- [Tempo Runbook (GitHub)](https://github.com/grafana/tempo/blob/main/operations/tempo-mixin/runbook.md)
- [Tempo Getting Started (Docker Example)](https://grafana.com/docs/tempo/latest/getting-started/docker-example/)
