# LGTM Observability Stack

Lokales Observability-Setup auf Basis des Grafana-Stacks. Kombiniert Metriken, Logs und Traces in einer einzigen Grafana-Oberfläche — vollständig mit Docker Compose betreibbar, ohne Cloud-Abhängigkeiten.

---

## Architektur

```
Anwendung / k6-tracing
        │
        ▼ OTLP (gRPC :4317 / HTTP :4318)
    ┌───────┐       Traces      ┌───────┐
    │ Alloy │ ────────────────▶│ Tempo │──┐
    └───────┘                   └───────┘  │ metrics_generator
        │ Logs (/var/log)                  ▼
        ▼                           ┌────────────┐
    ┌──────┐                        │ Prometheus │
    │ Loki │                        └────────────┘
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

Alloy übernimmt zwei Aufgaben und ersetzt Promtail:

- **Traces**: Empfängt OTLP (gRPC + HTTP) und leitet an Tempo weiter
- **Logs**: Liest `/var/log/*log` vom Host und sendet an Loki

Konfiguration: [shared/config.alloy](shared/config.alloy)

### Tempo Metrics Generator

Tempo generiert automatisch Metriken aus Traces und schreibt sie per Remote Write in Prometheus:
- **Service Graphs** — Abhängigkeiten und Latenz zwischen Services
- **Span Metrics** — Rate, Fehlerrate, Dauer pro Operation
- **Local Blocks** — Metriken-Abfragen direkt über TraceQL

### Datenpersistenz

Tempo-Daten werden in `./tempo-data` auf dem Host gespeichert. Loki und Prometheus speichern im Container (`/tmp/loki` bzw. intern). Beim `docker compose down -v` gehen Container-Volumes verloren — `./tempo-data` bleibt erhalten.

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

### Traces erkunden

1. http://localhost:3000 öffnen (kein Login erforderlich)
2. **Explore** → Datenquelle **Tempo** → Tab **Search** → **Run query**
3. Einen Trace auswählen um das Trace-Diagramm zu sehen
4. Tab **Service graph** für automatisch generierte Service-Maps (nach ~2 Minuten verfügbar)

### Logs erkunden

1. **Explore** → Datenquelle **Loki**
2. Label-Filter: `{job="varlogs"}` für System-Logs
3. Mit LogQL filtern, z.B.: `{job="varlogs"} |= "error"`

### Metriken erkunden

1. **Explore** → Datenquelle **Prometheus**
2. Tempo-generierte Metriken z.B.: `rate(traces_spanmetrics_calls_total[5m])`

### Traces mit Metriken verknüpfen (Exemplars)

Prometheus ist mit Exemplar-Support konfiguriert. In Grafana unter **Explore → Prometheus** auf den Exemplar-Punkt in einem Graph klicken um direkt zum zugehörigen Trace zu springen.

---

## Konfigurationsdateien

```
├── config/
│   ├── alertmanager.yaml   # Alertmanager Routen und Receiver
│   ├── loki-config.yaml    # Loki Storage, Schema, Pattern Ingester
│   └── tempo.yaml          # Tempo Distributor, Ingester, Metrics Generator
├── shared/
│   ├── config.alloy        # Alloy Pipeline (Traces + Logs)
│   ├── grafana-datasources.yaml  # Grafana Datasource-Provisioning
│   └── prometheus.yaml     # Prometheus Scrape-Config
└── docker-compose.yaml
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
