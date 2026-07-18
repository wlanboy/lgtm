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
| [kubernetes/istio-prometheus-tls.yaml](kubernetes/istio-prometheus-tls.yaml) | TLS-Origination für HTTPS-Prometheus (optional, siehe unten) |

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

Bei Änderungen an [kubernetes/configmap.yaml](kubernetes/configmap.yaml) oder [kubernetes/rbac.yaml](kubernetes/rbac.yaml) übernehmen die Alloy-Pods die neue Config nicht automatisch — `kubectl apply` ändert nur die Ressourcen, das DaemonSet-Pod-Template bleibt gleich. Reroll manuell erzwingen:

```bash
kubectl apply -f kubernetes/configmap.yaml
kubectl -n monitoring rollout restart daemonset/alloy
kubectl -n monitoring rollout status daemonset/alloy
```

---

## Helm-Chart

Alternativ zu den einzelnen YAML-Dateien unter [kubernetes/](kubernetes/) steht derselbe Deploy als Helm-Chart unter [helm/alloy-lgtm/](helm/alloy-lgtm/) zur Verfügung. Der Chart erzeugt exakt dieselben Ressourcen (Namespace, RBAC, ConfigMap, DaemonSet, optional Istio) und macht `<LGTM-HOST>` sowie Istio als Werte konfigurierbar statt per `sed`.

### Werte

| Key | Default | Bedeutung |
|---|---|---|
| `namespace` | `monitoring` | Ziel-Namespace |
| `lgtmHost` | `192.168.1.100` | Host des externen LGTM-Stacks |
| `alloy.image` | `grafana/alloy:v1.15.1` | Alloy-Image |
| `istio.enabled` | `true` | Namespace-Label `istio-injection: enabled`, Envoy-Metriken in der Alloy-Config, `Service/alloy-otlp` + `Telemetry`-Ressourcen |
| `istio.samplingPercentage` | `10` | Trace-Sampling-Rate |
| `istio.installMeshConfigPatch` | `false` | Deployt die ConfigMap `istio` in `istio-system` mit — **überschreibt bestehende `extensionProviders`**, vorher `kubectl get cm istio -n istio-system -o yaml` prüfen |

### Anwenden

```bash
helm install alloy helm/alloy-lgtm \
  --set lgtmHost=192.168.1.91 \
  --set istio.enabled=true

# Status prüfen
kubectl -n monitoring get pods -l app=alloy
```

Ohne Istio (z. B. Cluster ohne Service Mesh):

```bash
helm install alloy helm/alloy-lgtm \
  --set lgtmHost=192.168.1.91 \
  --set istio.enabled=false
```

### Aktualisieren

Bei Änderungen am Chart (z. B. RBAC, Config, Werte) reicht `helm upgrade` — der Namespace bleibt dabei erhalten, nur die geänderten Ressourcen werden aktualisiert:

```bash
helm upgrade alloy helm/alloy-lgtm \
  --set lgtmHost=192.168.178.91 \
  --set istio.enabled=true

# Status prüfen
helm status alloy -n monitoring
kubectl -n monitoring rollout restart daemonset/alloy
kubectl -n monitoring rollout status daemonset/alloy
```

Das DaemonSet trägt eine `checksum/config`-Annotation (Hash der ConfigMap-Templates). Ändert sich die Config über `--set`, ändert sich der Hash, und Kubernetes rollt die Pods automatisch neu — kein manueller Restart nötig.

`--install` kombiniert Install und Upgrade in einem Befehl (idempotent, z. B. für CI):

```bash
helm upgrade --install alloy helm/alloy-lgtm \
  --set lgtmHost=192.168.178.91 \
  --set istio.enabled=true
```

### Prometheus per TLS (Istio TLS-Origination)

Spricht der externe Prometheus HTTPS, muss Alloy dessen Zertifikat vertrauen. Statt den Trust in der Alloy-Config zu verdrahten (`tls_config` + gemountetes CA-Cert), übernimmt hier **Istio** die TLS-Origination: Alloy sendet weiterhin plain HTTP an `<LGTM-HOST>:9090` (unverändert in [kubernetes/configmap.yaml](kubernetes/configmap.yaml)), der istio-proxy Sidecar fängt den Traffic per `ServiceEntry`/`DestinationRule` ab, baut selbst die TLS-Verbindung auf und prüft das Zertifikat gegen eine CA aus einem Secret. Der Trust liegt damit vollständig in Istio, nicht in Alloy.

Voraussetzung: CA-Zertifikat des externen Prometheus als Secret im `monitoring`-Namespace (Key muss `ca.crt` heißen):

```bash
kubectl create secret generic prometheus-ca-cert -n monitoring --from-file=ca.crt=ca/ca.pem
```

**Raw Manifests:**

```bash
sed -i 's/<LGTM-HOST>/192.168.178.91/g' kubernetes/istio-prometheus-tls.yaml
kubectl apply -f kubernetes/istio-prometheus-tls.yaml
```

**Helm:**

```bash
helm upgrade --install alloy helm/alloy-lgtm \
  --set lgtmHost=192.168.178.91 \
  --set istio.enabled=true \
  --set istio.prometheusTls.enabled=true
```

Beide Varianten legen zusätzlich eine `Role`/`RoleBinding` an, die dem istio-proxy Sidecar (läuft unter der ServiceAccount `alloy`) per SDS lesenden Zugriff auf genau dieses eine Secret erlaubt — sonst kann Envoy die CA nicht laden.

**Verifikation:**

```bash
kubectl -n monitoring logs -l app=alloy -c istio-proxy --tail=50 | grep -i prometheus
kubectl -n monitoring logs -l app=alloy --tail=20 | grep -i "remote_write\|Failed to send batch"
```

Kein `Failed to send batch` mehr in den Alloy-Logs → die TLS-Origination funktioniert.

> **Hinweis:** Unabhängig vom TLS-Trust muss die Ziel-IP aus dem Cluster heraus überhaupt erreichbar sein (reine TCP-Verbindung, vor jeder TLS-Prüfung). Das lässt sich isoliert testen mit:
> ```bash
> POD=$(kubectl -n monitoring get pods -l app=alloy -o jsonpath='{.items[0].metadata.name}')
> kubectl -n monitoring exec "$POD" -c istio-proxy -- curl -sv --max-time 5 https://<LGTM-HOST>:9090/-/healthy
> ```
> Ein `Connection timed out` hier deutet auf ein Netzwerk-/Firewall-Problem zwischen Cluster und LGTM-Host hin, nicht auf TLS/Trust.

---

## Verifikation

Nach dem Deployment (egal ob per `kubectl apply` oder Helm) in drei Schritten prüfen: läuft das Deployment, greift Istio wie erwartet, kommen Daten im LGTM-Stack an.

### 1. Deployment-Check

```bash
# Helm-Release-Status (nur bei Helm-Deploy)
helm status alloy -n monitoring

# Namespace-Label prüfen
kubectl get ns monitoring --show-labels

# Läuft ein Alloy-Pod pro Node?
kubectl -n monitoring get pods -l app=alloy -o wide
kubectl get nodes --no-headers | wc -l

# Logs auf Fehler prüfen
kubectl -n monitoring logs -l app=alloy --tail=50
```

Alloy hat eine eigene UI mit Pipeline-Graph — Port-Forward und im Graph-Tab auf rot markierte (fehlerhafte) Komponenten achten:

```bash
kubectl -n monitoring port-forward svc/alloy-otlp 12345:12345
# http://localhost:12345 öffnen
```

### 2. Istio-Check

Nur relevant, wenn im Cluster tatsächlich eine Istio-Control-Plane installiert ist und `istio.enabled=true` gesetzt wurde.

```bash
# Konfigurationsfehler/Warnungen mesh-weit prüfen
istioctl analyze -n monitoring

# Sidecar-Injection prüfen: enthält ein Pod den Container "istio-proxy"?
kubectl -n monitoring get pods -o jsonpath='{.items[*].spec.containers[*].name}'

# Telemetry-Ressourcen vorhanden?
kubectl -n istio-system get telemetry
```

> **Hinweis:** Das Namespace-Label `istio-injection: enabled` gilt für den gesamten `monitoring`-Namespace — also auch für die Alloy-Pods selbst. Da Alloy per Host-Network die kubelet-API abfragt und `/var/log` direkt liest, ist ein Envoy-Sidecar in den Alloy-Pods meist nicht gewünscht. Falls das vermieden werden soll, `sidecar.istio.io/inject: "false"` als Pod-Annotation im DaemonSet ergänzen.

`istioctl analyze` meldet u. a. Warnungen zur Port-Namenskonvention (`IST0118`), wenn Service-Ports nicht mit einem erkannten Protokoll-Präfix beginnen (`grpc-`, `http-`, `tcp-`, …). Die Ports von `Service/alloy-otlp` heißen deshalb `grpc-otlp` (4317) und `http-otlp` (4318) — ohne korrektes Präfix behandelt Istio den Traffic als generisches TCP und es fehlt L7-Telemetrie/Routing für diesen Port.

### 3. Data-Check

- **Metriken:** Prometheus-Targets unter `http://<LGTM-HOST>:9090/targets` — Jobs `kubelet`, `cadvisor` und (bei aktiviertem Istio) `istio_envoy` müssen `UP` sein.
- **Envoy/Mesh-Metriken:** Grafana → Explore → Prometheus, z. B. `envoy_cluster_upstream_rq_total` abfragen — liefert erst Daten, sobald echter Traffic durch einen istio-injizierten App-Namespace läuft.
- **Logs:** Grafana → Explore → Loki:
  ```
  {namespace="monitoring", app="alloy"}
  ```
  bestätigt Alloys eigene Logs. Envoy Access-Logs (sobald `istio.installMeshConfigPatch=true` und `Telemetry/access-logs` aktiv sind) über `{namespace="istio-system"}` bzw. den jeweiligen App-Namespace abfragen.
- **Traces:** eine Anfrage durch einen Mesh-Workload schicken, danach Grafana → Explore → Tempo nach dem Service-Namen suchen. Kommt nichts an: Erreichbarkeit von `http://<LGTM-HOST>:4317` aus einem Pod im Cluster testen (`curl` oder `otel-cli`).

---

## Apps instrumentieren

Alloy läuft bereits — Entwickler müssen nur ihr Deployment entsprechend kennzeichnen. Keine zentrale Konfigurationsänderung nötig.

### Schritt 1: Label `app` setzen (Pflicht)

Das Label `app` ist die wichtigste Kennung in Loki und Prometheus. Ohne es lassen sich Logs und Metriken nicht sauber einem Service zuordnen.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  selector:
    matchLabels:
      app: my-app          # muss mit template.labels übereinstimmen
  template:
    metadata:
      labels:
        app: my-app        # wird als Label in Loki und Prometheus übernommen
```

### Schritt 2: Metriken freischalten (opt-in)

Annotations am Pod-Template eintragen — nicht am Deployment selbst:

```yaml
spec:
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        prometheus.io/scrape: "true"   # aktiviert Scraping
        prometheus.io/port: "8080"     # Port des Metrics-Endpoints
        prometheus.io/path: "/metrics" # optional, Default: /metrics
```

Danach sind Metriken in Grafana unter **Explore → Prometheus** abfragbar:
```
http_requests_total{app="my-app"}
```

### Schritt 3: Traces senden (opt-in)

Alloy läuft als DaemonSet auf jedem Node. Der OTLP-Endpunkt ist über `status.hostIP` erreichbar — kein Service-DNS nötig.

```yaml
spec:
  template:
    spec:
      containers:
        - name: my-app
          env:
            - name: NODE_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.hostIP
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://$(NODE_IP):4317"
            - name: OTEL_SERVICE_NAME
              value: "my-app"
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: "namespace=$(MY_NAMESPACE)"
            - name: MY_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
```

Unterstützte Protokolle: gRPC (`:4317`), HTTP (`:4318`). Alle gängigen OTel-SDKs (Java, Go, Python, Node.js, .NET) funktionieren ohne weitere Konfiguration.

### Schritt 4: Logs abfragen (kein Aufwand)

Logs werden automatisch gesammelt — solange die App nach **stdout/stderr** schreibt. Kein Log-Agent, kein Filebeat, keine Konfiguration.

In Grafana unter **Explore → Loki**:
```
{namespace="default", app="my-app"}
{namespace="default", app="my-app"} |= "ERROR"
{namespace="default", app="my-app"} | json | level="error"
```

### Vollständiges Deployment-Beispiel

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      containers:
        - name: my-app
          image: my-app:latest
          ports:
            - containerPort: 8080
          env:
            - name: NODE_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.hostIP
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://$(NODE_IP):4317"
            - name: OTEL_SERVICE_NAME
              value: "my-app"
```

### Übersicht: Was ist nötig?

| Ziel | Aufwand | Was tun |
|---|---|---|
| Logs in Loki | nichts | App nach stdout schreiben (Default) |
| Metriken in Prometheus | 2 Annotations | `prometheus.io/scrape: "true"` + Port |
| Traces in Tempo | 2 Env-Variablen | `NODE_IP` + `OTEL_EXPORTER_OTLP_ENDPOINT` |
| Alles filterbar | 1 Label | `app: my-app` am Pod setzen |

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
