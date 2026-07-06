# DOCKER.md — svc-observability

Guia Docker deste stack. Complementa `CLAUDE.md`. Para a visão end-to-end do monorepo,
ver `mondaha/docs/INFRA.md §16`.

Compose canônico: **`docker-compose.observability.yaml`** (o `docker-compose.yaml` está
**DEPRECATED**). Branch: `svc-observability-fs`.

## Comandos

```bash
# 1. subir o mondaha primeiro — ele é dono da obs_net e a cria automaticamente
#    (cd ../mondaha && docker compose up -d)
#    Só se rodar ESTE stack sem o mondaha: docker network create obs_net

# 2. subir o stack de observabilidade (só se anexa à obs_net já criada)
docker compose -f docker-compose.observability.yaml up -d

# logs (todos ou por serviço)
docker compose -f docker-compose.observability.yaml logs -f
docker compose -f docker-compose.observability.yaml logs -f prometheus

# status
docker compose -f docker-compose.observability.yaml ps

# derrubar (mantém volumes prometheus_data/grafana_data)
docker compose -f docker-compose.observability.yaml down

# derrubar + apagar volumes (perde histórico de métricas/dashboards)
docker compose -f docker-compose.observability.yaml down -v
```

## Serviços × imagem × porta × função

| Serviço | Imagem | Porta (`expose`) | Função |
|---|---|---|---|
| `prometheus` | `prom/prometheus:v2.55.0` | 9090 | scraping + TSDB (retention 30d, `--web.enable-lifecycle`) |
| `grafana` | `grafana/grafana-oss:11.1.0` | 3000 | dashboards; datasource Prometheus provisionado |
| `node-exporter` | `prom/node-exporter:v1.8.1` | 9100 | CPU/RAM/disk/FS do host (`pid: host`) |
| `cadvisor` | `gcr.io/cadvisor/cadvisor:v0.49.1` | 8080 | métricas por container (privileged) |
| `datadog` | `datadog/agent:7` | 8125/udp, 8126, 4317, 4318 | DogStatsD, APM, OTLP gRPC/HTTP |
| `postgres_exporter` | `prometheuscommunity/postgres-exporter` | 9187 | Postgres do mondaha (`postgres:5432`) |
| `redis_exporter` | `oliver006/redis_exporter:latest` | 9121 | Redis do mondaha (`redis:6379`) |

Todas as portas são `expose` (visíveis só dentro da `obs_net`) — **não** há `ports:`
publicando na VPS. Acesso externo é feito por reverse proxy (ver abaixo).

## Rede `obs_net` (só anexa; o mondaha é dono)

```yaml
networks:
  obs_net:
    external: true
    name: obs_net
```

`obs_net` é uma **bridge cross-projeto** que permite que containers de dois `docker compose`
distintos compartilhem um mesmo domínio de DNS interno. **O stack do mondaha é o dono**: seu
`docker-compose.yml` declara a rede como não-external (`networks: obs_net: { name: obs_net }`)
e a **cria automaticamente** no `docker compose up`. **Este** stack apenas **se anexa** a ela
(`external: true, name: obs_net`) — não a cria. Sem ela, o Prometheus/Datadog daqui não
conseguiriam resolver os containers do mondaha.

Por isso, **suba o mondaha primeiro** (ele cria a `obs_net`) e depois este stack. Só é preciso
`docker network create obs_net` se você rodar **este** stack **sem** o mondaha, já que ele é
`external`-only e não pode criar a rede.

## Como o Prometheus scrapeia os serviços do mondaha

O mondaha anexa `postgres`, `redis`, `api`, `app`, `svc-kg`, `svc-face-recon` à `obs_net`
(além da rede `default` dele). Assim o Prometheus resolve cada um por **nome de container
via DNS interno do Docker** e coleta `/metrics`:

| Job | Target | Origem |
|---|---|---|
| `mondaha-api` | `api:3001/metrics` | Elysia API |
| `svc-face-recon` | `svc-face-recon:8000/metrics` | reconhecimento facial (porta **8000**) |
| `svc-kg` | `svc-kg:8080/metrics` | knowledge graph |
| `postgres-exporter` | `postgres-exporter:9187` | exporter do Postgres |
| `redis-exporter` | `redis-exporter:9121` | exporter do Redis |
| `node-exporter` / `cadvisor` / `prometheus` | `node-exporter:9100` / `cadvisor:8080` / `prometheus:9090` | base (host, containers, self) |

`scrape_interval: 15s`. Editar targets em `prometheus/prometheus.yml`; espelhar no Datadog
em `datadog/openmetrics.d/conf.yaml`.

## Expor Grafana/Prometheus externamente

Como os serviços usam `expose` (não `ports`), não há publicação direta na VPS. Para acessar
de fora, vincular um domínio/reverse proxy (Coolify) ao serviço:

- **Grafana** → porta interna `3000` (UI principal; login via `GF_ADMIN_USER`/`GF_ADMIN_PASSWORD`).
- **Prometheus** → porta interna `9090` (opcional; proteger atrás de auth, expõe todas as métricas).

No Coolify: apontar o proxy do domínio para o container `grafana:3000` (ou `prometheus:9090`).

## Troubleshooting

- **`up` falha com "network obs_net declared as external, but could not be found"**: a rede
  ainda não existe porque o mondaha (dono da `obs_net`) não subiu. Suba o mondaha primeiro
  (`cd ../mondaha && docker compose up -d`); ou, ao rodar este stack sozinho, crie a rede à
  mão: `docker network create obs_net`.
- **`postgres-exporter`/`redis-exporter` sem dados / target DOWN**: o stack do mondaha não
  está `up` ou não foi anexado à `obs_net`. Subir o mondaha primeiro (ver ordem em `CLAUDE.md`).
- **`svc-face-recon` target DOWN / sem métricas**: confirmar porta **8000** (não 8080) em
  `prometheus/prometheus.yml` e `datadog/openmetrics.d/conf.yaml`. `svc-kg` usa 8080.
- **Grafana sem datasource**: checar `grafana/provisioning/datasources/prometheus.yaml`
  (`url: http://prometheus:9090`, `isDefault: true`).
- **Datadog sem métricas**: `DD_API_KEY` ausente/errado no `.env`; conferir `DD_SITE`.
```
