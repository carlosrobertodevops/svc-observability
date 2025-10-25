# svc-observability

## 1) docker-compose.observability.yaml

- docker-compose.observability.yaml

```yaml

name: observability

services:
  prometheus:
    image: prom/prometheus:v2.55.0
    container_name: prometheus
    restart: unless-stopped
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=30d
      - --web.enable-lifecycle
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    expose:
      - "9090"
    networks:
      - obs_net

  grafana:
    image: grafana/grafana-oss:11.1.0
    container_name: grafana
    restart: unless-stopped
    user: "472"               # evita permissões chatas em /var/lib/grafana
    environment:
      GF_SECURITY_ADMIN_USER: ${GF_ADMIN_USER:-admin}
      GF_SECURITY_ADMIN_PASSWORD: ${GF_ADMIN_PASSWORD:-changeme}
      GF_USERS_ALLOW_SIGN_UP: "false"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    expose:
      - "3000"               # no Coolify, informe 3000 como porta interna
    networks:
      - obs_net

  node-exporter:
    image: prom/node-exporter:v1.8.1
    container_name: node-exporter
    restart: unless-stopped
    pid: host
    command:
      - --path.rootfs=/rootfs
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
      - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    expose:
      - "9100"
    networks:
      - obs_net

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    container_name: cadvisor
    restart: unless-stopped
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    expose:
      - "8080"
    networks:
      - obs_net

  datadog:
    image: datadog/agent:7
    container_name: datadog
    restart: unless-stopped
    environment:
      DD_API_KEY: ${DD_API_KEY}         # defina no Coolify como Secret
      DD_SITE: ${DD_SITE:-datadoghq.com}
      DD_ENV: ${DD_ENV:-production}
      DD_SERVICE: observability
      DD_VERSION: "1.0.0"
      DD_DOGSTATSD_NON_LOCAL_TRAFFIC: "true"
      DD_LOGS_ENABLED: "true"
      DD_PROCESS_CONFIG_ENABLED: "true"
      DD_CONTAINER_COLLECT_ALL: "true"
      DD_APM_ENABLED: "true"             # ativa rastreamento APM (porta 8126)
      DD_OTLP_CONFIG_RECEIVER_PROTOCOLS_GRPC_ENDPOINT: "0.0.0.0:4317"
      DD_OTLP_CONFIG_RECEIVER_PROTOCOLS_HTTP_ENDPOINT: "0.0.0.0:4318"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /proc/:/host/proc/:ro
      - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro
      - ./datadog/openmetrics.d/conf.yaml:/etc/datadog-agent/conf.d/openmetrics.d/conf.yaml:ro
    expose:
      - "8125/udp"   # DogStatsD
      - "8126"       # APM
      - "4317"       # OTLP gRPC
      - "4318"       # OTLP HTTP
    networks:
      - obs_net


networks:
  obs_net:
    external: true

volumes:
  prometheus_data: {}
  grafana_data: {}

```

Como expor Grafana no Coolify: crie um “Application → Docker Compose”, cole o YAML, e no mapeamento de porta use 3000 (porta interna). Depois, vincule um domínio no Coolify para o serviço grafana.

⸻

## 2) prometheus/prometheus.yml

- prometheus/prometheus.yml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  scrape_timeout: 10s
  external_labels:
    env: production

scrape_configs:
  # O próprio Prometheus
  - job_name: prometheus
    static_configs:
      - targets: ["prometheus:9090"]

  # Métricas do host (CPU/RAM/Disk/FS)
  - job_name: node-exporter
    static_configs:
      - targets: ["node-exporter:9100"]

  # Métricas de containers Docker
  - job_name: cadvisor
    static_configs:
      - targets: ["cadvisor:8080"]

  # === SUAS APPS AQUI (exemplos) ===
  # Para cada microserviço que expõe /metrics, adicione um bloco:
  - job_name: svc-face-recon
    metrics_path: /metrics
    static_configs:
      - targets: ["svc-face-recon:8080"]   # ajuste a porta real
        labels:
          app: svc-face-recon

  - job_name: svc-kg
    metrics_path: /metrics
    static_configs:
      - targets: ["svc-kg:8080"]           # ajuste a porta real
        labels:
          app: svc-kg

```

Dica: para o Prometheus enxergar suas apps, elas precisam estar na mesma rede obs_net (veja o item 4 abaixo). Ajuste as portas/paths conforme cada serviço.

⸻

## 3) Provisionar o datasource do Grafana (auto)

- grafana/provisioning/datasources/prometheus.yaml

```yaml
apiVersion: 1
deleteDatasources:
  - name: Prometheus
    orgId: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
```

Depois, no Grafana, você pode importar dashboards prontos (ex.: “Node Exporter Full”, “cAdvisor”, etc.) pelo ID do Grafana.com.

⸻

## 4) Datadog: scraping dos mesmos endpoints (opcional, mas recomendado)

- datadog/openmetrics.d/conf.yaml

```yaml
init_config:

instances:
  # Coleta do node-exporter
  - openmetrics_endpoint: http://node-exporter:9100/metrics
    namespace: node_exporter
    metrics:
      - "*"

  # Coleta do cAdvisor
  - openmetrics_endpoint: http://cadvisor:8080/metrics
    namespace: cadvisor
    metrics:
      - "*"

  # Suas apps (ajuste portas e paths)
  - openmetrics_endpoint: http://svc-face-recon:8080/metrics
    namespace: svc_face_recon
    metrics:
      - "*"

  - openmetrics_endpoint: http://svc-kg:8080/metrics
    namespace: svc_kg
    metrics:
      - "*"
```

Defina DD_API_KEY como Secret no Coolify. Se quiser enviar traces por OTLP, aponte seus serviços para <http://datadog:4317> (gRPC) ou <http://datadog:4318> (HTTP).

⸻

5) Como plugar seus microserviços na observabilidade

Em cada Compose de app (no Coolify), acrescente:

```yaml
networks:
  obs_net:
    external: true

services:
  sua-app:
    # ...
    networks:
      - obs_net
```

Assim, prometheus e datadog (que estão na obs_net) conseguem resolver sua-app por DNS interno e coletar /metrics.

⸻

6) Passo-a-passo rápido no Coolify

 1. New Application → Docker Compose → cole o docker-compose.observability.yaml.
 2. Crie os arquivos de config pelos Mounts ou pela aba de Editor (se usar Git, suba o repositório com essa estrutura).
 3. Em Environment/Secrets, defina:
 • GF_ADMIN_USER, GF_ADMIN_PASSWORD
 • DD_API_KEY (obrigatório), DD_SITE (se usar datadoghq.eu, por ex.)
 4. Deploy. Depois, abra o serviço grafana e vincule um domínio/porta 3000.
 5. Em cada microserviço, conecte à obs_net e adicione o job correspondente no prometheus.yml.

Pronto! Você terá:
 • Grafana com dashboards em cima do Prometheus (host + containers + suas apps).
 • Datadog Agent enviando métricas/Logs/APM da VPS e containers para sua conta.

Se quiser, eu já te envio um dashboard base do Grafana (JSON) com painéis para node_exporter + cAdvisor e exemplos de painéis para /metrics das suas APIs.
