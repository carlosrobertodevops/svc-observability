# CLAUDE.md — svc-observability

Instruções operacionais concisas para agentes trabalharem neste repo. Não duplicar a
documentação do mondaha; para a visão end-to-end ver `mondaha/docs/INFRA.md §16`.

## Propósito

Stack de observabilidade **acoplado ao monorepo mondaha** via **rede Docker externa
compartilhada `obs_net`**. Coleta métricas de host, containers e microserviços do mondaha
(Prometheus + Datadog) e as visualiza no Grafana. Roda em `docker-compose` próprio,
separado do stack do mondaha; os dois se enxergam por DNS interno graças à `obs_net`.

Branch de trabalho: **`svc-observability-fs`**.

## Compose canônico

- ✅ **`docker-compose.observability.yaml`** — ÚNICO compose a usar. Todos os services em
  `obs_net` (external) e portas via `expose` (não `ports`); acesso externo só via reverse
  proxy (Coolify).
- ⛔ `docker-compose.yaml` — **DEPRECATED** (header marcado). Não usar/editar.

## Stack e portas (todas via `expose`, sem publish direto)

| Serviço | Imagem | Porta(s) | Função |
|---|---|---|---|
| prometheus | `prom/prometheus:v2.55.0` | 9090 | TSDB + scraping (retention 30d) |
| grafana | `grafana/grafana-oss:11.1.0` | 3000 | dashboards (datasource Prometheus) |
| node-exporter | `prom/node-exporter:v1.8.1` | 9100 | métricas do host |
| cadvisor | `gcr.io/cadvisor/cadvisor:v0.49.1` | 8080 | métricas de containers |
| datadog | `datadog/agent:7` | 8125/udp, 8126, 4317, 4318 | DogStatsD, APM, OTLP gRPC/HTTP |
| postgres_exporter | `prometheuscommunity/postgres-exporter` | 9187 | Postgres do mondaha |
| redis_exporter | `oliver006/redis_exporter` | 9121 | Redis do mondaha |

Sem Loki/Tempo/OTEL-collector standalone — OTLP entra pelo Datadog agent (4317/4318).

## Targets de scrape

`prometheus/prometheus.yml` (`scrape_interval: 15s`), jobs:
`prometheus:9090`, `node-exporter:9100`, `cadvisor:8080`, `svc-face-recon:8000/metrics`,
`svc-kg:8080/metrics`, `mondaha-api` (`api:3001/metrics`), `postgres-exporter:9187`,
`redis-exporter:9121`.

Datadog espelha os mesmos endpoints em `datadog/openmetrics.d/conf.yaml`.

## Env vars (`.env`, ver `.env.example`)

- Grafana: `GF_ADMIN_USER`, `GF_ADMIN_PASSWORD`
- Datadog: `DD_API_KEY` (obrigatório), `DD_SITE=datadoghq.com`, `DD_ENV=production`

## Ordem de subida

1. `docker network create obs_net` (a rede precisa existir antes de qualquer stack).
2. Subir o **mondaha** (`cd ../mondaha && docker compose up -d`) — anexa
   postgres/redis/api/app/svc-kg/svc-face-recon à `obs_net`.
3. Subir **este stack** (`docker compose -f docker-compose.observability.yaml up -d`).

Os exporters `postgres-exporter`/`redis-exporter` só alcançam `postgres:5432`/`redis:6379`
porque o mondaha os anexa à `obs_net`; ambos os stacks precisam estar `up`.

## Detalhe de porta (bug corrigido)

`svc-face-recon` escuta em **8000** (não 8080). Endpoint correto:
`http://svc-face-recon:8000/metrics` — tanto em `prometheus/prometheus.yml` quanto em
`datadog/openmetrics.d/conf.yaml`. `svc-kg` usa 8080 (correto).

Ver também `DOCKER.md` para comandos e troubleshooting.
