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

Konfiguration: [config/tempo.yaml](config/tempo.yaml) — Beachte: `block_retention: 24h` und `max_block_duration: 5m` sind bewusst niedrig für den Demo-/Dev-Betrieb gesetzt. In Produktion deutlich höher konfigurieren.

Dieser Stack läuft als **Tempo-Monolith** (`storage.trace.backend: local`, keine separaten Microservices, kein Kafka-basiertes Ingestion). Teile der offiziellen Tempo-Doku beziehen sich auf die Microservices-Architektur (`live-store`, `block-builder`, Kafka-Konsumenten) ab Tempo 3.x — diese Hinweise sind für dieses Setup **nicht relevant** und wurden unten bewusst weggelassen.

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
   - Sampling-Konfiguration in der App/SDK prüfen (z. B. `parentbased_traceidratio`) — der Standard-Sampler vieler SDKs samplet nur 1:1000, was in Dev-Umgebungen leicht als "keine Traces" missverstanden wird
   - Bei `k6-tracing`: läuft der Container? `docker compose logs k6-tracing`

2. **Kommen Spans bei Alloy an?**
   - Alloy UI (http://localhost:12345) → Komponente für OTLP-Receiver prüfen
   - Alloy-Metriken abfragen (über Prometheus, Datasource "prometheus", oder direkt `curl -s http://localhost:12345/metrics`):
     ```promql
     rate(receiver_accepted_spans_ratio_total[5m])   # otelcol.receiver.otlp — sollte > 0 sein
     rate(receiver_refused_spans_ratio_total[5m])    # sollte 0 sein
     rate(exporter_sent_spans_ratio_total[5m])       # otelcol.exporter.otlp — sollte > 0 sein
     rate(exporter_send_failed_spans_ratio_total[5m]) # sollte 0 sein
     ```
     `refused`/`send_failed` > 0 bedeutet: Alloy verwirft Spans oder kann sie nicht an Tempo weiterleiten. Bleiben alle vier Metriken bei `0`, kommen bei Alloy überhaupt keine Spans an → Ursache liegt bei Schritt 1 (App) oder Netzwerk/Endpunkt.

3. **Kommen Spans bei Tempo an (Distributor)?**
   - Tempo-Metrik prüfen:
     ```promql
     rate(tempo_distributor_spans_received_total[5m])  # sollte > 0 sein
     rate(tempo_receiver_refused_spans[5m])             # sollte 0 sein
     rate(tempo_discarded_spans_total[5m])              # sollte 0 sein, Label "reason" zeigt Grund
     ```
   - Ist `tempo_distributor_spans_received_total` dauerhaft `0`, prüfen ob Protokoll/Port stimmt (Schritt 4) — noch nichts kommt beim Distributor an.
   - Bei Rate-Limiting loggt der Distributor exakt:
     ```
     rpc error: code = ResourceExhausted desc = RATE_LIMITED: ingestion rate limit (30000000 bytes) exceeded while adding 10 bytes
     ```
     → `overrides:` Limits (`max_bytes_per_trace`, `max_traces_per_user`, Ingestion-Rate) prüfen/erhöhen.
   - Distributor-Logs auf `pusher failed to consume trace data" err="context canceled"` prüfen → deutet meist auf einen Client-seitigen Verbindungsabbruch hin, nicht auf ein Tempo-Problem.
   - Für Detail-Debugging temporär `distributor.log_discarded_spans.enabled: true` (optional `include_all_attributes: true`) in `config/tempo.yaml` setzen — Tempo loggt dann pro verworfenem Span `msg=discarded spanid=... traceid=...` inkl. Grund. Nach dem Debuggen wieder deaktivieren (Log-Volumen!).

4. **Falsches Protokoll/Port?**
   - OTLP gRPC (`4317`) vs. OTLP HTTP (`4318`) nicht verwechseln — häufigste Fehlerquelle bei neu instrumentierten Apps.
   - Container-interne Anwendungen müssen an `tempo:4317`/`alloy:4317` senden (Service-Name im Docker-Netzwerk), nicht `localhost`.

5. **Sind die Traces schon wieder abgelaufen?**
   - `block_retention: 24h` in diesem Stack → alte Traces sind nach 24 Stunden weg. Bei "Trace war vorgestern da, heute nicht mehr" ist das meist die Ursache, kein Bug.

6. **Grafana-Datasource-Problem?**
   - Tempo-Datasource-URL muss auf den **HTTP-Port 3200** zeigen, nicht auf gRPC (9095) — Grafana leitet die gRPC-Verbindung intern vom HTTP-Endpunkt ab.
   - Bei TLS-Fehlern: Zertifikat-Mismatch zwischen Grafana und Tempo prüfen (in diesem Stack standardmäßig kein TLS, also nur relevant bei eigenen Anpassungen).

---

## 3. Weitere bekannte Fehlerbilder

| Symptom | Wahrscheinliche Ursache | Maßnahme |
|---|---|---|
| Tempo-Container startet nicht / crasht sofort | Volume-Permission auf `tempo-data` (Tempo läuft als User `10001`) | Prüfen ob `init`-Container erfolgreich lief: `docker compose logs init`. Ggf. `docker compose up -d init` erneut ausführen |
| `rpc error: ... RATE_LIMITED: ingestion rate limit (...) exceeded` beim Senden | Ingestion-Rate-Limit im Distributor erreicht | Ingestion-Rate in `overrides:` erhöhen, oder Sende-Last reduzieren (Sampling in der App). Diagnose: `distributor.log_discarded_spans.enabled: true` setzen für Details pro Span |
| `error using pageFinder` bei Suche (HTTP 500), genauer: `error querying store in Querier.FindTraceByID: error using pageFinder...` | Korrupter/inkonsistenter Block im lokalen Storage (z. B. unvollständiger Schreibvorgang) | Betroffenen Block identifizieren (Pfad-Schema `<tenant>/<block-id>` unter `/var/tempo/blocks`), Tempo-Container stoppen, Block-Verzeichnis aus dem Volume löschen, Container neu starten. **Datenverlust nur für diesen Block** — bereits im WAL vorhandene, noch nicht kompaktierte Daten bleiben erhalten |
| `queue doesn't have room for 100 jobs` / `failed to add a job to work queue` bei der Compaction | Compactor-Queue-Kapazität erschöpft relativ zur Workload (viele/große Blöcke) | Metriken prüfen: `tempodb_compaction_bytes_written_total` (>0 = Worker arbeitet), `tempodb_compaction_errors_total` (>0 = Fehler in Compactor-Logs suchen). Danach `storage.trace.pool.queue_depth` (Default reicht selten für Lastspitzen) und `compactor.compaction.max_block_bytes`/`max_compaction_objects` anpassen |
| `too many jobs in the queue` / HTTP 429 bei Suche | Query-Frontend-Kapazität erschöpft (viele parallele/teure Suchen) | Zeitraum der Suche eingrenzen, `duration_slo`/`throughput_bytes_slo` in `query_frontend` prüfen, Suchlast reduzieren |
| Service-Name-/Span-Name-Dropdown in Grafana Explore leer ("No options found", kein Fehler) | Tag-Value-Query überschreitet `max_bytes_per_tag_values_query`-Limit → Tempo liefert stillschweigend ein leeres Ergebnis statt eines Fehlers (Endpunkt `/api/search/tag/service.name/values`) | Anzahl/Kardinalität der abgefragten Tag-Werte reduzieren; `max_bytes_per_tag_values_query` in `overrides:` erhöhen (Richtwert ~50MB) |
| `500 Internal Server Error: response larger than the max (<size> vs <limit>)` | gRPC-Message-Size-Limit überschritten, meist zwischen Querier und Query-Frontend bei großen Suchergebnissen | Je nach betroffener Komponente erhöhen: `query_frontend.max_grpc_streaming_packet_size` (Default 2 MiB) für Streaming-Antworten, `server.grpc_server_max_recv_msg_size`/`grpc_server_max_send_msg_size` (gilt pro Prozess, nicht global synchronisiert — ggf. in mehreren Komponenten setzen), `querier.frontend_worker.grpc_client_config.max_send_msg_size`; bei OTLP-Ingest zusätzlich `distributor.receivers.otlp.protocols.grpc.max_recv_msg_size_mib` prüfen |
| Service-Graph/Span-Metrics fehlen in Grafana | Metrics-Generator schreibt nicht erfolgreich zu Prometheus | `docker compose logs tempo \| grep -i "remote write"`; prüfen ob Prometheus mit `--web.enable-remote-write-receiver` läuft (ist in diesem Stack Standard); ~2 Minuten Vorlaufzeit nach Start abwarten |
| Grafana zeigt "no data" nur bei Service Graph | Zu kurze Wartezeit nach Deploy, oder Edges laufen ab bevor der passende Gegenpart ankommt | Service Graphs brauchen laut Doku ca. 2 Minuten Anlaufzeit. Bei anhaltendem Problem Metrik `tempo_metrics_generator_processor_service_graphs_expired_edges` prüfen — hohe Rate bedeutet, `metrics_generator.processor.service_graphs.wait` (Default 10s) ist zu knapp bemessen |
| Span-Metrics/Service-Graph-Kardinalität explodiert (viele Zeitreihen in Prometheus) | Hochkardinale Labels wie `span_name` oder `url` aus dem Metrics-Generator (in diesem Stack aktiv: `processors: [service-graphs, span-metrics, local-blocks]`) | Rate prüfen mit `sum by (tenant, label_name) (rate(tempo_metrics_generator_registry_label_values_limited_total[5m]))`; `overrides.defaults.metrics_generator.max_cardinality_per_label` setzen (überzählige Werte werden zu `__cardinality_overflow__`), alternativ `span_name_sanitization: dry_run` testen und dann aktivieren |
| Trace-Suche zeigt inkonsistente Ergebnisse bei langlaufenden Traces | Trace erstreckt sich über mehrere Blöcke (durch `max_block_duration: 5m`) — TraceQL berücksichtigt bei Spanset-Operatoren laut Doku nur den zusammenhängenden Trace-Anteil im aktuellen Block | Erwartetes Verhalten bei kurzem `max_block_duration`; für Produktion höher konfigurieren. Zusätzlich Metrik `tempo_warnings_total{reason="disconnected_trace_flushed_to_wal"}` bzw. `rootless_trace_flushed_to_wal` beobachten — deutet auf Spans mit fehlendem Parent bzw. fehlendem Root-Span hin (oft Instrumentierungsfehler) |
| Hoher Speicherverbrauch / OOM bei Tempo | Einzelne sehr große Span-Attribute (>2048 Byte, Default von `max_attribute_bytes`), sehr lange/große Traces (100K–1M Spans) oder hochkardinale große Attribute | `max_attribute_bytes` in `overrides` setzen (Attribute werden dann getruncated, sichtbar über Metrik `tempo_distributor_attributes_truncated_total`); `max_bytes_per_trace` pro Tenant in `overrides` begrenzen (Empfehlung: mit 15MB starten, max. 60MB); alternativ große Attribute schon im OTel Collector/Alloy per Attribute-Processor entfernen |

Hinweis: Die offizielle Doku beschreibt zusätzlich Consumer-Lag-Probleme (`tempo_ingest_group_partition_lag`) zwischen Distributor und `live-store`/`block-builder`. Das betrifft nur Tempo-3.x-Deployments mit Kafka-basierter Ingestion und ist für diesen Docker-Compose-Monolithen **nicht relevant**.

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

## 5. Kubernetes: Traces aus dem Cluster fehlen

Bezug: [kubernetes.md](kubernetes.md). Apps im Cluster senden Traces **nicht** an Tempo direkt, sondern an den lokalen Alloy-Pod auf ihrem Node (`$(NODE_IP):4317`), der sie an den externen Tempo (Docker-Host) weiterleitet.

**Checkliste:**

1. **Laufen die Alloy-DaemonSet-Pods auf allen Nodes?**
   ```bash
   kubectl -n monitoring get pods -l app=alloy -o wide
   ```
   Ein Pod pro Node erwartet. Fehlt einer, sendet keine App auf diesem Node erfolgreich Traces (Tolerations sind bereits auf `operator: Exists` gesetzt, decken also alle Taints ab — Pending-Pods deuten eher auf Ressourcen- oder Image-Pull-Probleme hin).

2. **Ist der `<LGTM-HOST>`-Platzhalter in der ConfigMap tatsächlich ersetzt?**
   ```bash
   kubectl -n monitoring get cm alloy-config -o yaml | grep -i "4317\|otlp"
   ```
   Größter Stolperstein laut Deployment-Anleitung: Wird `<LGTM-HOST>` vergessen zu ersetzen, landet nichts beim Tempo-Backend.

3. **Setzt die App wirklich `NODE_IP` (`status.hostIP`) statt einer festen Adresse?**
   ```bash
   kubectl exec <app-pod> -- env | grep -E "NODE_IP|OTEL_EXPORTER_OTLP_ENDPOINT"
   ```
   Eine hartkodierte Adresse funktioniert nur zufällig auf einem Node und erklärt "Traces kommen nur von manchen Pods an".

4. **Netzwerkpfad Alloy-Pod → Tempo-Host offen?**
   ```bash
   kubectl -n monitoring exec -it <alloy-pod> -- nc -zv <LGTM-HOST> 4317
   ```
   Firewall/Security-Group zwischen Cluster-Nodes und Docker-Host prüfen (analog zum Prometheus-Remote-Write-Check in [troubleshooting-prometheus.md](troubleshooting-prometheus.md)).

5. **Alloy-Pod-Logs auf OTLP-Fehler prüfen:**
   ```bash
   kubectl -n monitoring logs -l app=alloy --tail=200 | grep -i "otlp\|refused\|failed"
   ```

6. **Bei Istio-Integration:** Prüfen, ob der MeshConfig-Patch tatsächlich angewendet wurde und nicht durch eine spätere Änderung überschrieben ist:
   ```bash
   kubectl get cm istio -n istio-system -o yaml | grep -A5 extensionProviders
   kubectl get telemetry -n istio-system
   ```

**Bekannte Fehlerbilder (Kubernetes-spezifisch):**

| Symptom | Wahrscheinliche Ursache | Maßnahme |
|---|---|---|
| Gar keine Traces aus dem Cluster | `<LGTM-HOST>` in `configmap.yaml` nicht ersetzt | ConfigMap prüfen, `kubectl apply -f kubernetes/configmap.yaml`, Alloy-Pods rollen (`kubectl rollout restart daemonset/alloy -n monitoring`) |
| Traces nur von einzelnen Pods/Nodes | App nutzt feste Adresse statt `$(NODE_IP):4317` | Deployment auf `status.hostIP`-Pattern umstellen (siehe [kubernetes.md](kubernetes.md#schritt-3-traces-senden-opt-in)) |
| Istio-Mesh-Traces fehlen trotz funktionierender App-Traces | MeshConfig-Patch nicht aktiv oder überschrieben | `extensionProviders` in der Istio-ConfigMap erneut setzen, DaemonSet-Rollout abwarten |
| Alloy-Pod auf einem Node crasht/restart-loop | Portkonflikt auf 4317/4318/12345 mit anderem Prozess des Nodes | `kubectl -n monitoring describe pod <alloy-pod>`, belegte Ports auf dem Node prüfen |

---

## 6. Eskalationskriterien

- Datenverlust durch korrupte Blöcke oder versehentliches `docker compose down -v`
- Anhaltende `tempo_receiver_refused_spans` trotz angepasster Limits → Kapazitätsproblem, Skalierung prüfen
- OOM-Kills des Tempo-Containers unter Last → Ressourcenlimits und Ingestion-Rate gemeinsam mit Team abstimmen

---

## Links

Die relevanten Fehlerbilder aus der offiziellen Doku sind oben in Abschnitt 2/3 bereits inhaltlich eingearbeitet. Als Ausgangspunkt für Themen, die über dieses Runbook hinausgehen (z. B. Tempo-3.x-Microservices-Betrieb mit Kafka):

- [Kubernetes-Anbindung dieses Stacks](kubernetes.md)
- [Troubleshoot Tempo (offizielle Doku, Index)](https://grafana.com/docs/tempo/latest/troubleshooting/)
- [Troubleshoot Tempo data source in Grafana](https://grafana.com/docs/grafana/latest/datasources/tempo/troubleshooting/)
- [Tempo Runbook (GitHub)](https://github.com/grafana/tempo/blob/main/operations/tempo-mixin/runbook.md)
