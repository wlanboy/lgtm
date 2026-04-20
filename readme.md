# LGTM Observability Stack

Lokales Observability-Setup auf Basis des Grafana-Stacks. Kombiniert Metriken, Logs und Traces in einer einzigen Grafana-Oberfläche — vollständig mit Docker Compose betreibbar, ohne Cloud-Abhängigkeiten.

---

## Architektur

```
Anwendung / k6-tracing
        │
        ▼ OTLP (gRPC :4317 / HTTP :4318)
    ┌───────┐       Traces      ┌───────┐
    │ Alloy │ ─────────────────▶│ Tempo │──┐
    └───────┘                   └───────┘  │ metrics_generator
        │ Logs (/var/log)                  ▼
        │                            ┌────────────┐
        │                            │ Prometheus │
        ▼                            └────────────┘
    ┌──────┐                              │
    │ Loki │                              │
    └──────┘                              │
        │                                 │
        └──────────┬──────────────────────┘
                   ▼
              ┌─────────┐
              │ Grafana │
              └─────────┘
                   │
                   ▼
            ┌─────────────┐
            │ Alertmanager│
            └─────────────┘
```

---

## Komponenten

| Service | Image | Version | Beschreibung |
|---|---|---|---|
| **Grafana** | `grafana/grafana` | 13.0.1 | Visualisierung, Dashboards, Explore |
| **Loki** | `grafana/loki` | 3.7.1 | Log-Aggregation und -Abfrage |
| **Tempo** | `grafana/tempo` | 2.10.4 | Distributed Tracing Backend |
| **Prometheus** | `prom/prometheus` | v3.10.0 | Metriken-Scraping und -Speicherung |
| **Alloy** | `grafana/alloy` | v1.15.1 | OTel Collector: OTLP→Tempo, Logs→Loki |
| **Alertmanager** | `prom/alertmanager` | v0.32.0 | Alert-Routing und -Benachrichtigung |
| **Memcached** | `memcached` | 1.6 | Cache für Tempo Frontend-Search |
| **k6-tracing** *(optional)* | `xk6-client-tracing` | v0.0.9 | Synthetischer Trace-Generator |

### k6-tracing (optional)

`k6-tracing` sendet kontinuierlich synthetische Traces an Tempo und ist nur für Demo- und Testzwecke gedacht. Damit sind sofort Traces in Grafana sichtbar, ohne eine eigene Anwendung instrumentieren zu müssen.

**Eigene Anwendungen senden Traces?** Dann `k6-tracing` aus `docker-compose.yaml` entfernen — sonst erzeugt es unnötigen Lärm in den Trace-Daten.

### Alloy als zentraler Collector

Alloy übernimmt mehrere Aufgaben und ersetzt Promtail:

- **Traces**: Empfängt OTLP (gRPC + HTTP) und leitet an Tempo weiter
- **Logs**: Liest `/var/log/*log` vom Host und sendet an Loki
- **Host Metriken**: Erfasst Node-Metriken (entspricht node_exporter) und sendet an Prometheus

Konfiguration: [shared/config.alloy](shared/config.alloy)

### Tempo Metrics Generator

Tempo generiert automatisch Metriken aus Traces und schreibt sie per Remote Write in Prometheus:
- **Service Graphs** — Abhängigkeiten und Latenz zwischen Services
- **Span Metrics** — Rate, Fehlerrate, Dauer pro Operation
- **Local Blocks** — Metriken-Abfragen direkt über TraceQL

### Datenpersistenz

Alle Daten werden in benannten Docker Volumes gespeichert und überleben `docker compose down`:

| Volume | Service | Inhalt |
|---|---|---|
| `tempo-data` | Tempo | Trace-Daten |
| `loki-data` | Loki | Log-Daten |
| `prometheus-data` | Prometheus | Metriken-Daten |
| `grafana-data` | Grafana | Dashboards, Einstellungen |
| `alertmanager-data` | Alertmanager | Silences, Notification-Log |

**Achtung:** `docker compose down -v` löscht alle Volumes unwiderruflich.

---

## URLs

| Service | URL | Beschreibung |
|---|---|---|
| **Grafana** | http://localhost:3000 | Haupt-UI: Dashboards, Explore |
| **Grafana Traces** | http://localhost:3000/a/grafana-traces-app | Trace Explorer |
| **Prometheus** | http://localhost:9090 | PromQL, Targets, Alerts |
| **Alertmanager** | http://localhost:9093 | Alert-Routing UI |
| **Alloy UI** | http://localhost:12345 | Pipeline-Graph, Debug |
| **Tempo API** | http://localhost:3200 | Tempo HTTP API |
| **Loki API** | http://localhost:3100 | Loki HTTP API |
| **Loki Ready** | http://localhost:3100/ready | Readiness Check |
| **Loki Metrics** | http://localhost:3100/metrics | Loki interne Metriken |

---

## Start / Stop

```bash
# Stack starten
docker compose up -d

# Status prüfen
docker compose ps

# Logs eines Service anzeigen
docker compose logs -f alloy

# Stack stoppen (Volumes behalten)
docker compose down

# Stack stoppen und alle Volumes löschen
docker compose down -v
```

---

## Daten senden

### Traces via OTLP

Eigene Anwendungen können Traces direkt an Alloy senden:

```
OTLP gRPC:  localhost:4317
OTLP HTTP:  localhost:4318
```

Alternativ direkt an Tempo (ohne Alloy-Pipeline):
```
Jaeger HTTP:    localhost:14268
Zipkin:         localhost:9411
```

### Logs

Alloy liest automatisch alle `*.log`-Dateien aus `/var/log` des Hosts und schickt sie an Loki. Für eigene Log-Pfade [shared/config.alloy](shared/config.alloy) anpassen:

```alloy
local.file_match "my_app" {
  path_targets = [{
    __path__ = "/var/log/myapp/*.log",
    job      = "myapp",
  }]
}
```

---

## Grafana verwenden

### Dashboards

Beim Start werden folgende Dashboards automatisch provisioniert (keine manuelle Installation nötig):

| Dashboard | Inhalt |
|---|---|
| **Node Exporter Full** | CPU, RAM, Disk, Netzwerk des Hosts |
| **Docker cAdvisor** | Container-Metriken (CPU, RAM, Restarts) |
| **Loki Dashboard** | Log-Volumen, Fehlerrate, Log-Streams |
| **Tempo / Traces** | Trace-Latenz, Service-Map, Error-Rate |
| **Alertmanager** | Alert-Status, Silences, Gruppen |
| **Kubernetes Cluster** | Pods, Deployments, Nodes (bei K8s-Anbindung) |
| **Kubernetes Pods** | Pod-Ressourcen per Namespace (bei K8s-Anbindung) |
| **Istio Mesh** | Service-Mesh Traffic, Fehlerrate (bei Istio) |

Dashboard-JSONs liegen unter [shared/dashboards/](shared/dashboards/).

### Traces erkunden

1. http://localhost:3000 öffnen (kein Login erforderlich)
2. **Explore** → Datenquelle **Tempo** → Tab **Search** → **Run query**
3. Einen Trace auswählen um das Trace-Diagramm zu sehen
4. Tab **Service graph** für automatisch generierte Service-Maps (nach ~2 Minuten verfügbar)

### Logs erkunden

1. **Explore** → Datenquelle **Loki**
2. Label-Filter: `{job="varlogs"}` für System-Logs, `{container="alloy"}` für Docker-Logs
3. Mit LogQL filtern, z.B.: `{job="varlogs"} |= "error"`

### Metriken erkunden

1. **Explore** → Datenquelle **Prometheus**
2. Tempo-generierte Metriken z.B.: `rate(traces_spanmetrics_calls_total[5m])`
3. Container-Metriken z.B.: `container_cpu_usage_seconds_total`

### Metrik Targets überprüfen
1. http://localhost:9090/targets öffnen

### Traces mit Metriken verknüpfen (Exemplars)

Prometheus ist mit Exemplar-Support konfiguriert. In Grafana unter **Explore → Prometheus** auf den Exemplar-Punkt in einem Graph klicken um direkt zum zugehörigen Trace zu springen.

---

## Alerting

Alerts werden in Prometheus als Regeln definiert und über Alertmanager geroutet.

### Alert-Regel anlegen

Neue Datei `config/alerts.yaml` anlegen:

```yaml
groups:
  - name: example
    rules:
      - alert: HighErrorRate
        expr: rate(traces_spanmetrics_calls_total{status_code="STATUS_CODE_ERROR"}[5m]) > 0.1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Hohe Fehlerrate bei {{ $labels.service }}"
          description: "Fehlerrate: {{ $value | humanizePercentage }}"

      - alert: ContainerDown
        expr: absent(container_last_seen{name!=""})
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Container nicht erreichbar: {{ $labels.name }}"
```

Dann in [shared/prometheus.yaml](shared/prometheus.yaml) einbinden:

```yaml
rule_files:
  - /etc/prometheus/alerts.yaml
```

Und in [docker-compose.yaml](docker-compose.yaml) das Volume mounten:

```yaml
prometheus:
  volumes:
    - ./config/alerts.yaml:/etc/prometheus/alerts.yaml
```

### Alertmanager konfigurieren

Empfänger und Routen in [config/alertmanager.yaml](config/alertmanager.yaml) definieren:

```yaml
global:
  smtp_smarthost: 'localhost:25'

route:
  receiver: default
  group_by: [alertname, severity]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: oncall

receivers:
  - name: default
    email_configs:
      - to: 'team@example.com'
        from: 'alertmanager@example.com'

  - name: oncall
    webhook_configs:
      - url: 'http://oncall-service/webhook'
```

### Alert-Status prüfen

- Aktive Alerts: http://localhost:9090/alerts
- Routing und Silences: http://localhost:9093

---

## Kubernetes

Für die Anbindung eines Kubernetes-Clusters an diesen Stack siehe [kubernetes.md](kubernetes.md).

Kurzzusammenfassung: Grafana Alloy wird als DaemonSet im Cluster deployed und sendet Logs, Metriken und Traces an den laufenden Docker-Stack. Vier YAML-Dateien unter [kubernetes/](kubernetes/) sind deployfertig vorbereitet.

---

## Konfigurationsdateien

```
├── config/
│   ├── alertmanager.yaml       # Alertmanager Routen und Receiver
│   ├── loki-config.yaml        # Loki Storage, Schema, Pattern Ingester
│   └── tempo.yaml              # Tempo Distributor, Ingester, Metrics Generator
├── shared/
│   ├── config.alloy            # Alloy Pipeline (Traces, Logs, Docker, Host-Metriken)
│   ├── dashboards/             # Provisionierte Grafana Dashboard JSONs
│   ├── grafana-dashboards.yaml # Grafana Dashboard-Provisioning
│   ├── grafana-datasources.yaml# Grafana Datasource-Provisioning
│   └── prometheus.yaml         # Prometheus Scrape-Config
├── kubernetes/
│   ├── namespace.yaml          # Namespace monitoring
│   ├── rbac.yaml               # ServiceAccount, ClusterRole
│   ├── configmap.yaml          # Alloy-Config für Kubernetes
│   ├── configmap-with-istio.yaml # Alloy-Config inkl. Istio
│   ├── daemonset.yaml          # Alloy DaemonSet
│   └── istio.yaml              # Istio Telemetry + Service (optional)
└── docker-compose.yaml
```

---

## Troubleshooting

### Grafana zeigt "No data"

1. Alloy UI prüfen: http://localhost:12345 → Pipeline-Graph auf Fehler prüfen
2. Prometheus Targets prüfen: http://localhost:9090/targets → alle Targets `UP`?
3. Loki Readiness: `curl http://localhost:3100/ready` → muss `ready` zurückgeben
4. Container-Logs prüfen: `docker compose logs -f <service>`

### Loki startet nicht / bleibt unhealthy

```bash
docker compose logs loki
# Häufige Ursache: Berechtigungsproblem auf loki-data Volume
docker compose down -v && docker compose up -d
```

### Tempo-Traces fehlen nach Neustart

Traces liegen im `tempo-data` Volume — dieses überlebt `docker compose down`. Bei `docker compose down -v` gehen sie verloren. Volume-Status prüfen:

```bash
docker volume ls | grep lgtm
docker volume inspect lgtm_tempo-data
```

### Alloy sammelt keine Docker-Logs

Docker-Socket muss für Alloy erreichbar sein:

```bash
# Prüfen ob Socket gemountet ist
docker compose exec alloy ls -la /var/run/docker.sock

# Alloy neu starten
docker compose restart alloy
```

### Alertmanager sendet keine Benachrichtigungen

```bash
# Alertmanager-Konfiguration validieren
docker compose exec alertmanager amtool check-config /etc/alertmanager/alertmanager.yaml

# Aktive Alerts anzeigen
docker compose exec alertmanager amtool alert query
```

### Prometheus Remote Write schlägt fehl (Kubernetes)

Prometheus muss mit `--web.enable-remote-write-receiver` gestartet sein (bereits konfiguriert). Firewall-Regel prüfen: Cluster-Nodes müssen Port `9090` des LGTM-Hosts erreichen können.

```bash
# Vom Kubernetes-Node aus testen
curl -v http://<LGTM-HOST>:9090/-/healthy
```

---

## Links

- [Grafana Stack Übersicht](https://grafana.com/about/grafana-stack/)
- [Alloy Dokumentation](https://grafana.com/docs/alloy/latest/)
- [Loki Docker Setup](https://grafana.com/docs/loki/latest/setup/install/docker/)
- [Tempo Getting Started](https://grafana.com/docs/tempo/latest/getting-started/docker-example/)
- [Tempo Instrumentation](https://grafana.com/docs/tempo/latest/getting-started/instrumentation/)
- [Prometheus Remote Write](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write)
- [OpenTelemetry Protokoll](https://opentelemetry.io/docs/specs/otlp/)
