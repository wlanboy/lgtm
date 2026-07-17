# Troubleshooting: Prometheus (Metriken)

Incident-Runbook für Prometheus `v3.10.0` in diesem Stack. Ergänzt die Kurzsektion in [readme.md](readme.md#troubleshooting) um eine tiefergehende Analyse für Produktionsprobleme.

Relevante Endpunkte in diesem Setup:

| Zweck | URL/Port |
|---|---|
| Web-UI / PromQL | http://localhost:9090 |
| Health-Check | http://localhost:9090/-/healthy |
| Ready-Check | http://localhost:9090/-/ready |
| Targets-Status | http://localhost:9090/targets |
| Aktive Alerts | http://localhost:9090/alerts |
| Remote-Write-Empfang (von Tempo Metrics-Generator) | http://localhost:9090/api/v1/write |

Besonderheiten dieses Stacks: gestartet mit `--web.enable-remote-write-receiver` (Tempo schreibt Service-Graph-/Span-Metriken direkt hinein) und `--enable-feature=exemplar-storage,native-histograms`. Konfiguration: [shared/prometheus.yaml](shared/prometheus.yaml).

---

## 1. Sofort-Diagnose (erste 5 Minuten)

- [ ] `curl -s http://localhost:9090/-/healthy` und `/-/ready` → beide müssen `200 OK` liefern
- [ ] http://localhost:9090/targets → sind **alle** Targets `UP`? (prometheus, tempo, loki, alloy)
- [ ] `docker compose ps prometheus` → Restart-Count prüfen (`docker inspect --format='{{.RestartCount}}' <container>`)
- [ ] `docker compose logs --tail=200 prometheus` → auf `level=error`, `corruption`, `OOM` scannen
- [ ] http://localhost:9090/alerts → unerwartet viele `firing`/`pending` Alerts?
- [ ] `docker stats prometheus` → Speicher-/CPU-Auslastung im Blick

---

## 2. Incident-Checkliste: "Hoher Speicherverbrauch / OOM-Kill"

Die mit Abstand häufigste Prometheus-Produktionsstörung ist **Cardinality-getriebener Speicherverbrauch**.

1. **OOM bestätigen:**
   ```bash
   docker inspect prometheus --format='{{.State.OOMKilled}} restarts={{.RestartCount}}'
   ```
   Steigender Restart-Count in kurzen Abständen → klassisches OOM-Crash-Loop-Muster.

2. **Cardinality messen** (wichtigste Kennzahl: `prometheus_tsdb_head_series`):
   ```promql
   prometheus_tsdb_head_series
   ```
   Werte im Bereich weniger Zehntausend sind normal für einen kleinen Stack wie diesen. Werte im Millionen-Bereich sind ein klares Cardinality-Problem — jede Serie kostet ca. 1–3 KB RAM, d. h. 5 Mio. Serien benötigen allein für die TSDB 5–15 GB RAM.

3. **Ursache identifizieren** — welche Metrik erzeugt die meisten Serien:
   ```promql
   topk(10, count by (__name__)({__name__=~".+"}))
   ```
   Oder pro Label, um das "kardinalitätstreibende" Label zu finden:
   ```promql
   topk(10, count by (job)({__name__=~".+"}))
   ```
   In diesem Stack besonders relevant: Tempo-generierte `traces_spanmetrics_*`-Metriken können bei sehr granularen Span-Namen/Attributen schnell viele Serien erzeugen.

4. **Sofortmaßnahme, wenn die Quelle nicht schnell gefunden wird:**
   Diagnose-Instanz mit kurzer Retention parallel starten, um ohne Risiko für die Produktionsinstanz zu analysieren:
   ```bash
   docker run --rm -it prom/prometheus:v3.10.0 --storage.tsdb.retention.time=2h ...
   ```

5. **Dauerhaft beheben:**
   - Hochkardinale Labels an der Quelle entfernen (Instrumentierung/Relabeling), nicht im PromQL wegfiltern
   - `metric_relabel_configs` in [shared/prometheus.yaml](shared/prometheus.yaml) nutzen, um unnötige Label-Werte vor dem Ingest zu droppen
   - Alert einrichten, der frühzeitig warnt:
     ```yaml
     - alert: PrometheusHighCardinality
       expr: prometheus_tsdb_head_series > 1000000
       for: 10m
     ```
   - Regelmäßig (z. B. monatlich) `promtool tsdb analyze` gegen das `prometheus-data`-Volume laufen lassen, um langsam wachsende Cardinality früh zu erkennen

---

## 3. Incident-Checkliste: "Prometheus startet nicht / Crash-Loop nach Neustart"

Typisches Muster bei einer großen, unsauber beendeten TSDB: **WAL-Replay** beim Start dauert zu lange oder schlägt fehl.

1. **Logs auf WAL-Probleme prüfen:**
   ```bash
   docker compose logs prometheus | grep -i "wal\|corrupt"
   ```
2. **Automatische Reparatur ist Standardverhalten:** Prometheus versucht, ein korruptes WAL-Segment ab dem Fehlerpunkt zu kürzen und danach weiterzumachen (dabei gehen ggf. Daten aus dem betroffenen Segment verloren). Das wird beim Start geloggt und erhöht `prometheus_tsdb_wal_corruptions_total`.
3. **Wenn der Start selbst zu lange dauert** (kein Fehler, aber Container wird vom Orchestrator wegen Startup-/Healthcheck-Timeout getötet, bevor der WAL-Replay fertig ist): Healthcheck- bzw. Startzeit-Grenzen erhöhen — großer WAL kann mehrere Stunden Replay-Zeit benötigen.
4. **Manuelles Eingreifen (letzte Instanz):** Betroffene WAL-Segment-Dateien aus den Logs identifizieren und gezielt aus dem `prometheus-data`-Volume entfernen. Nur wenn automatische Reparatur fehlschlägt — vorher Volume sichern:
   ```bash
   docker run --rm -v lgtm_prometheus-data:/data -v $(pwd)/backup:/backup busybox tar czf /backup/prometheus-data.tgz /data
   ```
5. **Ressourcen für den Replay prüfen:** Der WAL-Replay benötigt kurzfristig mehr Speicher als der stationäre Betrieb. Wenn Container-Limits nur für Steady-State dimensioniert sind, schlägt der Replay fehl.

---

## 4. Incident-Checkliste: "Langsame Queries / Dashboards laden nicht"

- [ ] Query-Dauer prüfen: `prometheus_engine_query_duration_seconds` (p95). Werte > 30s deuten auf zu komplexe Queries für das Datenvolumen oder eine zu klein dimensionierte Instanz hin
- [ ] Recording-Rule-Auswertung prüfen:
  ```promql
  prometheus_rule_evaluation_duration_seconds
  ```
  Wenn der p99-Wert die `evaluation_interval` (hier: 15s, siehe [shared/prometheus.yaml](shared/prometheus.yaml)) übersteigt, fällt Prometheus mit der Regel-Auswertung zurück
- [ ] Teure Dashboard-Queries (lange Zeiträume, hohe Auflösung, viele Serien) durch Recording Rules vorverdichten
- [ ] Prüfen, ob gleichzeitig ein Cardinality-Problem vorliegt (Abschnitt 2) — Symptome überschneiden sich häufig

---

## 5. Weitere bekannte Fehlerbilder

| Symptom | Wahrscheinliche Ursache | Maßnahme |
|---|---|---|
| Target auf `/targets` dauerhaft `DOWN` | Zielservice nicht erreichbar oder falscher Port in `shared/prometheus.yaml` | Container-Status des Ziels prüfen, `docker compose exec prometheus wget -qO- http://<target>:<port>/metrics` |
| Tempo-Metriken (Service-Graph, Span-Metrics) fehlen | Remote-Write vom Tempo Metrics-Generator schlägt fehl | `docker compose logs tempo \| grep -i "remote write"`; prüfen ob `--web.enable-remote-write-receiver` aktiv ist (Standard in diesem Stack) |
| Remote-Write von Kubernetes-Nodes schlägt fehl | Firewall/Netzwerk zwischen Cluster und LGTM-Host blockiert Port 9090 | `curl -v http://<LGTM-HOST>:9090/-/healthy` vom Node aus testen (siehe [readme.md](readme.md)) |
| Alerts feuern nicht trotz erfüllter Bedingung | Regel nicht geladen oder `for`-Dauer noch nicht erreicht | http://localhost:9090/alerts prüfen (Status `pending` vs. `firing`); `rule_files` in `shared/prometheus.yaml` kontrollieren |
| Exemplare fehlen im Grafana-Graph | `exemplar-storage`-Feature nicht aktiv oder App liefert keine Exemplare | Feature-Flag in `docker-compose.yaml` prüfen (ist Standard aktiv); Instrumentierung der App auf Exemplar-Support prüfen |
| Disk-Alarm auf `prometheus-data` | Retention zu hoch für verfügbaren Speicher oder Cardinality-Explosion | `--storage.tsdb.retention.time`/`--storage.tsdb.retention.size` setzen, Cardinality-Analyse (Abschnitt 2) |

---

## 6. Nützliche Kommandos & PromQL-Snippets

```bash
# Health- und Statusendpunkte
curl -s http://localhost:9090/-/healthy
curl -s http://localhost:9090/-/ready
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'

# Live-Logs verfolgen
docker compose logs -f prometheus

# Ressourcennutzung live beobachten
docker stats prometheus

# TSDB analysieren (Cardinality-Hotspots finden)
docker compose exec prometheus promtool tsdb analyze /prometheus

# Konfiguration validieren, bevor sie live geht
docker compose exec prometheus promtool check config /etc/prometheus.yaml
```

PromQL für Incident-Analyse:

```promql
# Cardinality-Übersicht
prometheus_tsdb_head_series

# Top 10 Metriken nach Serienanzahl
topk(10, count by (__name__)({__name__=~".+"}))

# Scrape-Fehler pro Job
up == 0

# Ingestion-Rate (Samples/Sekunde)
rate(prometheus_tsdb_head_samples_appended_total[5m])

# WAL-Korruptionen seit Start
prometheus_tsdb_wal_corruptions_total
```

---

## 7. Kubernetes: Cluster-Metriken fehlen

Bezug: [kubernetes.md](kubernetes.md). Dieser Stack betreibt **kein** In-Cluster-Prometheus — der Alloy-DaemonSet-Pod pro Node scraped kubelet (`:10250`), cAdvisor (`/metrics/cadvisor`) und annotierte App-Pods lokal und schreibt die Daten per `remote_write` direkt in den externen Prometheus. Das heißt auch: **K8s-Targets erscheinen nicht unter `/targets`** — dort steht nur der eigene `prometheus`/`tempo`/`loki`/`alloy`-Scrape aus [shared/prometheus.yaml](shared/prometheus.yaml). Für Kubernetes-Metriken muss stattdessen über PromQL geprüft werden, ob aktuelle Serien ankommen.

**Checkliste:**

1. **Kommen überhaupt Kubernetes-Metriken per Remote-Write an?**
   ```promql
   # Alter der letzten Node-Metrik (sollte wenige Sekunden/Minuten sein)
   time() - max(node_time_seconds)
   # oder: existieren kubelet/cadvisor-Serien überhaupt?
   count(up{job=~".*kubelet.*|.*cadvisor.*"})
   ```
   Fehlen Serien komplett: `<LGTM-HOST>`-Platzhalter in [kubernetes/configmap.yaml](kubernetes/configmap.yaml) prüfen — das ist laut Deployment-Anleitung die häufigste Ursache.
   ```bash
   kubectl -n monitoring get cm alloy-config -o yaml | grep -i "9090\|remote_write"
   ```

2. **RBAC für kubelet/cAdvisor-Scrape korrekt?**
   Die ClusterRole erlaubt `get` auf die `nonResourceURLs` `/metrics` und `/metrics/cadvisor` sowie `get`/`list`/`watch` auf `nodes`, `nodes/metrics`, `nodes/proxy`:
   ```bash
   kubectl auth can-i get /metrics/cadvisor --as=system:serviceaccount:monitoring:alloy
   ```

3. **Alloy-Pod-Logs auf Scrape- oder Remote-Write-Fehler prüfen:**
   ```bash
   kubectl -n monitoring logs -l app=alloy --tail=200 | grep -i "remote_write\|kubelet\|cadvisor\|x509"
   ```
   `x509`/TLS-Fehler deuten auf das selbstsignierte kubelet-Zertifikat hin — je nach Cluster muss die Alloy-Konfiguration `insecure_skip_verify` oder das CA-Bundle des Clusters nutzen.

4. **Netzwerkpfad Alloy-Pod → Prometheus-Host offen?**
   ```bash
   kubectl -n monitoring exec -it <alloy-pod> -- nc -zv <LGTM-HOST> 9090
   ```
   Gleicher Check wie beim manuellen Remote-Write-Test in [readme.md](readme.md#prometheus-remote-write-schlägt-fehl-kubernetes), nur aus dem Pod statt vom Node ausgeführt.

5. **App-eigene Metriken fehlen trotz erwarteter Annotation?**
   Häufigster Fehler: Annotation steht am `Deployment` statt am Pod-Template.
   ```bash
   kubectl get pod <pod> -o jsonpath='{.metadata.annotations}'
   ```
   Muss `prometheus.io/scrape: "true"` und `prometheus.io/port` enthalten (siehe [kubernetes.md](kubernetes.md#schritt-2-metriken-freischalten-opt-in)).

6. **Istio Envoy-Metriken fehlen?**
   Prüfen, ob `configmap-with-istio.yaml` (statt der Standard-ConfigMap) angewendet wurde und Sidecar-Port `15090` je Pod erreichbar ist:
   ```bash
   kubectl exec <pod> -c istio-proxy -- curl -s localhost:15090/stats/prometheus | head
   ```

**Bekannte Fehlerbilder (Kubernetes-spezifisch):**

| Symptom | Wahrscheinliche Ursache | Maßnahme |
|---|---|---|
| Gar keine Cluster-Metriken (Node/Container) | `<LGTM-HOST>` in `configmap.yaml` nicht ersetzt | ConfigMap korrigieren, Alloy-DaemonSet neu ausrollen |
| Nur Node-, keine Container-Metriken | cAdvisor-`nonResourceURL`-Recht fehlt oder Pfad falsch | RBAC (`rbac.yaml`) und Scrape-Pfad in `configmap.yaml` prüfen |
| App-Metriken fehlen trotz Annotation | Annotation am Deployment statt am Pod-Template gesetzt | Annotation unter `spec.template.metadata.annotations` verschieben |
| TLS-/x509-Fehler in Alloy-Logs beim kubelet-Scrape | Selbstsigniertes kubelet-Zertifikat nicht vertraut | TLS-Konfiguration des Scrape-Targets in `configmap.yaml` anpassen |
| Doppelte/kurzzeitig inkonsistente Serien bei DaemonSet-Rollout | Alter und neuer Alloy-Pod kurzzeitig beide aktiv | Erwartetes, temporäres Verhalten — kein Handlungsbedarf |
| Istio-Envoy-Metriken fehlen | `configmap-with-istio.yaml` nicht angewendet oder Sidecar-Port nicht erreichbar | Konfiguration gemäß [kubernetes.md](kubernetes.md#istio-optional) anwenden |

---

## 8. Eskalationskriterien

- Wiederholte OOM-Kills trotz identifizierter und bereinigter Cardinality-Quelle
- WAL-Korruption, die automatische Reparatur nicht beheben kann und Datenverlust droht
- Remote-Write-Ausfälle, die Alerting-Fähigkeit (Alertmanager-Routing) beeinträchtigen

---

## Links

- [Kubernetes-Anbindung dieses Stacks](kubernetes.md)
- [Troubleshooting Common Prometheus Pitfalls (Last9)](https://last9.io/blog/troubleshooting-common-prometheus-pitfalls-cardinality-resource-utilization-and-storage-challenges/)
- [High Cardinality in Prometheus: How to Find and Fix It (Last9)](https://last9.io/blog/how-to-manage-high-cardinality-metrics-in-prometheus/)
- [Prometheus WAL Replay: When Your Metrics Database Can't Start Fast Enough](https://blog.landryzetam.net/posts/prometheus-wal-replay/)
- [Prometheus went OOM potentially because of expensive query (GitHub Discussion)](https://github.com/prometheus/prometheus/discussions/14397)
- [Prometheus Remote Write Konfiguration (offizielle Doku)](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write)
