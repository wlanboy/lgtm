# Kubernetes Logs & Metriken in den LGTM-Stack

Diese Anleitung beschreibt, wie Logs, Metriken und Traces aus einem Kubernetes-Cluster in den lokal laufenden LGTM-Stack (Loki, Grafana, Tempo, Prometheus) gebracht werden. Als Collector-Komponente wird **Grafana Alloy** als DaemonSet eingesetzt — damit läuft ein Alloy-Pod auf jedem Node und sammelt alle Daten lokal ein, bevor sie per Push an den externen Stack gesendet werden.

Kein separater Prometheus, kein Promtail, kein Jaeger-Agent — Alloy übernimmt alle drei Rollen in einer einzigen Komponente.

## Architektur

```
┌─────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                     │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │  Node 1  │  │  Node 2  │  │  Node 3  │                   │
│  │          │  │          │  │          │                   │
│  │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │                   │
│  │ │Alloy │ │  │ │Alloy │ │  │ │Alloy │ │  ← DaemonSet      │
│  │ │      │ │  │ │      │ │  │ │      │ │    (1x pro Node)  │
│  │ └──┬───┘ │  │ └──┬───┘ │  │ └──┬───┘ │                   │
│  │    │     │  │    │     │  │    │     │                   │
│  │ Pod-Logs │  │ Pod-Logs │  │ Pod-Logs │  ← /var/log       │
│  │ kubelet  │  │ kubelet  │  │ kubelet  │  ← :10250         │
│  │ cAdvisor │  │ cAdvisor │  │ cAdvisor │  ← :10250/cadvis  │
│  └──────────┘  └──────────┘  └──────────┘                   │
│       │               │              │                      │
└───────┼───────────────┼──────────────┼──────────────────────┘
        │               │              │
        └───────────────┴──────────────┘
                        │
              ┌─────────┴──────────┐
              │                    │
    ┌─────────▼──────┐   ┌────────▼────────┐
    │   OTLP :4317   │   │  remote_write   │
    │   OTLP :4318   │   │  :9090          │
    └────────┬───────┘   └────────┬────────┘
             │                    │
  ┌──────────▼──────────────────────────────────┐
  │              LGTM-Stack (Docker)            │
  │                                             │
  │  ┌─────────┐  ┌────────────┐  ┌──────────┐  │
  │  │  Tempo  │  │ Prometheus │  │   Loki   │  │
  │  │ :4317   │  │   :9090    │  │  :3100   │  │
  │  └────┬────┘  └─────┬──────┘  └────┬─────┘  │
  │       │             │              │        │
  │  ┌────┴─────────────┴──────────────┴─────┐  │
  │  │              Grafana :3000            │  │
  │  └───────────────────────────────────────┘  │
  └─────────────────────────────────────────────┘
```

### Datenfluss

| Datentyp | Quelle | Collector | Ziel |
|---|---|---|---|
| Pod-Logs | Kubernetes API (alle Container) | `loki.source.kubernetes` | Loki |
| Node-Metriken | kubelet `/metrics` | `prometheus.scrape` | Prometheus |
| Container-Metriken | kubelet `/metrics/cadvisor` | `prometheus.scrape` | Prometheus |
| App-Metriken | Pods mit Annotation | `prometheus.scrape` | Prometheus |
| Traces | Apps via OTLP | `otelcol.receiver.otlp` | Tempo |

---

## Vorgehen

### Warum Alloy als DaemonSet?

Alloy läuft als **ein Pod pro Node**. Damit hat jeder Alloy-Pod direkten Zugriff auf:
- die lokale kubelet-API (`NODE_IP:10250`) für Node- und Container-Metriken
- `/var/log` des Hosts für Node-System-Logs
- die Kubernetes-API für Pod-Log-Streaming aller Container auf dem Node

Alternativ zu einem eigenen Prometheus im Cluster wird hier **remote_write** genutzt: Alloy scraped die Metriken und schreibt sie direkt in den externen Prometheus-Stack. Das vermeidet eine doppelte Prometheus-Instanz.

### RBAC

Alloy benötigt Leserechte auf die Kubernetes-API, um Pod- und Service-Metadaten für Labels (namespace, pod, app) abzufragen. Der ClusterRole werden nur `get`, `list`, `watch` Rechte erteilt — keine Schreibrechte.

### Pod-Logs

`loki.source.kubernetes` streamt Logs direkt über die Kubernetes-API (kein Dateisystem-Zugriff nötig). Labels wie `namespace`, `pod`, `container` und `app` werden automatisch aus den Pod-Metadaten befüllt und sind in Grafana/Loki sofort filterbar.

### App-Metriken per Annotation

Pods werden nur dann gescraped, wenn sie die Annotation `prometheus.io/scrape: "true"` tragen. Port und Pfad sind konfigurierbar. Das erlaubt opt-in pro Applikation ohne zentrale Konfigurationsänderung.

### Traces

Alloy öffnet einen OTLP-Receiver (gRPC :4317, HTTP :4318) auf jedem Node. Apps referenzieren den Node per `status.hostIP` und senden Traces direkt an den lokalen Alloy, der sie an Tempo weiterleitet.

---

## Deployment

### Voraussetzungen

- LGTM-Stack läuft und ist vom Cluster aus erreichbar
- `<LGTM-HOST>` in [kubernetes/configmap.yaml](kubernetes/configmap.yaml) ersetzen (2x Loki, Prometheus, Tempo)

### Dateien

| Datei | Inhalt |
|---|---|
| [kubernetes/namespace.yaml](kubernetes/namespace.yaml) | Namespace `monitoring` |
| [kubernetes/rbac.yaml](kubernetes/rbac.yaml) | ServiceAccount, ClusterRole, ClusterRoleBinding |
| [kubernetes/configmap.yaml](kubernetes/configmap.yaml) | Alloy-Konfiguration (Logs, Metriken, Traces) |
| [kubernetes/configmap-with-istio.yaml](kubernetes/configmap-with-istio.yaml) | Alloy-Konfiguration inkl. Istio Envoy-Metriken (optional) |
| [kubernetes/daemonset.yaml](kubernetes/daemonset.yaml) | Alloy DaemonSet (1x pro Node) |
| [kubernetes/istio.yaml](kubernetes/istio.yaml) | Istio-Integration (optional, siehe unten) |

### Anwenden

```bash
# 1. LGTM-Host in der ConfigMap setzen
sed -i 's/<LGTM-HOST>/192.168.1.100/g' kubernetes/configmap.yaml

# 2. Reihenfolge einhalten (Namespace zuerst, dann RBAC, dann Workloads)
kubectl apply -f kubernetes/namespace.yaml
kubectl apply -f kubernetes/rbac.yaml
kubectl apply -f kubernetes/configmap.yaml
kubectl apply -f kubernetes/daemonset.yaml

# 3. Status prüfen
kubectl -n monitoring get pods -l app=alloy
kubectl -n monitoring logs -l app=alloy --tail=50
```

---

## Apps instrumentieren

### Metriken (opt-in per Annotation)

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"   # optional, Default: /metrics
```

### Traces via OTLP

```yaml
env:
  - name: NODE_IP
    valueFrom:
      fieldRef:
        fieldPath: status.hostIP
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://$(NODE_IP):4317"
```

### Logs in Grafana abfragen

Keine App-Änderung nötig. Alle Pod-Logs sind sofort in Loki verfügbar:

```
{namespace="default", app="my-app"}
{namespace="default", pod=~"my-app-.*", container="main"}
```

---

## Istio (optional)

Wenn Istio im Cluster installiert ist, liefert es drei zusätzliche Datenquellen ohne App-Änderungen:

| Datentyp | Quelle | Ergebnis |
|---|---|---|
| Envoy-Metriken | Sidecar Port 15090 `/stats/prometheus` | Request-Rate, Latenz, Fehlerrate pro Service in Prometheus |
| Distributed Traces | Istio Telemetry API -> Alloy OTLP | Vollständige Service-Mesh-Traces in Tempo |
| Access Logs | Envoy stdout (JSON) | HTTP-Zugriffslog aller Sidecar-Verbindungen in Loki |

### Vorgehen

**1. Istio MeshConfig patchen** — registriert Alloy als Tracing-Provider:

```bash
kubectl patch configmap istio -n istio-system --type merge -p '
{
  "data": {
    "mesh": "extensionProviders:\n- name: alloy-otlp\n  opentelemetry:\n    service: alloy-otlp.monitoring.svc.cluster.local\n    port: 4317\n"
  }
}'
```

> Achtung: Dieser Patch überschreibt bestehende `extensionProviders`. Bei vorhandener Mesh-Config erst lesen (`kubectl get cm istio -n istio-system -o yaml`) und manuell zusammenführen.

**2. Istio-Ressourcen und ConfigMap anwenden:**

```bash
kubectl apply -f kubernetes/istio.yaml
kubectl apply -f kubernetes/configmap-with-istio.yaml
kubectl rollout restart daemonset/alloy -n monitoring
```

Damit werden angelegt:
- `Service/alloy-otlp` im Namespace `monitoring` — macht Alloy clusterweit als OTLP-Endpunkt erreichbar
- `Telemetry/tracing` in `istio-system` — aktiviert 100 % Sampling mesh-weit
- `Telemetry/access-logs` in `istio-system` — schreibt Envoy Access Logs nach stdout (werden von Loki eingesammelt)

**3. Sampling anpassen** (optional):

```yaml
# kubernetes/istio.yaml — Telemetry/tracing
randomSamplingPercentage: 10  # 10 % für Produktion empfohlen
```

---

## Zusammenfassung

| Was | Wie | Ergebnis |
|---|---|---|
| Pod-Logs aller Container | `loki.source.kubernetes` per Kubernetes-API | In Loki, filterbar nach namespace/pod/container/app |
| Node- & Container-Metriken | kubelet + cAdvisor per HTTPS scrape | In Prometheus unter `node_*`, `container_*` |
| App-eigene Metriken | Annotation `prometheus.io/scrape: "true"` | In Prometheus, opt-in pro Pod |
| Traces | OTLP an lokalen Alloy-Pod (`NODE_IP:4317`) | In Tempo, verknüpfbar mit Logs via TraceID |
| Istio Envoy-Metriken | Sidecar Port 15090 (automatisch) | Service-Mesh-Metriken in Prometheus |
| Istio Traces | Telemetry API -> Alloy OTLP | Mesh-weites Tracing ohne App-Änderung |
| Alle Daten | Grafana als zentrales UI | Dashboards, Alerts, Explore über eine Oberfläche |

**4 YAML-Dateien**, **1 `sed`-Befehl** für den Host, **fertig.**

Der Stack ist bewusst schlank gehalten: kein In-Cluster-Prometheus, keine separate Log-Pipeline, kein eigener Trace-Collector. Alloy als DaemonSet ist der einzige Kubernetes-seitige Baustein — der Rest läuft im bestehenden Docker-Stack.
