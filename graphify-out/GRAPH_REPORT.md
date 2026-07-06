# Graph Report - svc-observability  (2026-07-06)

## Corpus Check
- 1 files · ~1,591 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 9 nodes · 8 edges · 2 communities
- Extraction: 100% EXTRACTED · 0% INFERRED · 0% AMBIGUOUS
- Token cost: 0 input · 0 output

## Graph Freshness
- Built from commit: `68ffdc94`
- Run `git rev-parse HEAD` and compare to check if the graph is stale.
- Run `graphify update .` after code changes (no API cost).

## Community Hubs (Navigation)
- [[_COMMUNITY_Community 0|Community 0]]
- [[_COMMUNITY_Community 1|Community 1]]

## God Nodes (most connected - your core abstractions)
1. `svc-observability` - 6 edges
2. `7) Integração com mondaha` - 3 edges
3. `1) docker-compose.observability.yaml` - 1 edges
4. `2) prometheus/prometheus.yml` - 1 edges
5. `3) Provisionar o datasource do Grafana (auto)` - 1 edges
6. `4) Datadog: scraping dos mesmos endpoints (opcional, mas recomendado)` - 1 edges
7. `Bridge `obs_net`` - 1 edges
8. `Jobs de scrape (mondaha)` - 1 edges

## Surprising Connections (you probably didn't know these)
- None detected - all connections are within the same source files.

## Import Cycles
- None detected.

## Communities (2 total, 0 thin omitted)

### Community 0 - "Community 0"
Cohesion: 0.33
Nodes (5): 1) docker-compose.observability.yaml, 2) prometheus/prometheus.yml, 3) Provisionar o datasource do Grafana (auto), 4) Datadog: scraping dos mesmos endpoints (opcional, mas recomendado), svc-observability

### Community 1 - "Community 1"
Cohesion: 0.67
Nodes (3): 7) Integração com mondaha, Bridge `obs_net`, Jobs de scrape (mondaha)

## Knowledge Gaps
- **6 isolated node(s):** `1) docker-compose.observability.yaml`, `2) prometheus/prometheus.yml`, `3) Provisionar o datasource do Grafana (auto)`, `4) Datadog: scraping dos mesmos endpoints (opcional, mas recomendado)`, `Bridge `obs_net`` (+1 more)
  These have ≤1 connection - possible missing edges or undocumented components.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `svc-observability` connect `Community 0` to `Community 1`?**
  _High betweenness centrality (0.893) - this node is a cross-community bridge._
- **Why does `7) Integração com mondaha` connect `Community 1` to `Community 0`?**
  _High betweenness centrality (0.464) - this node is a cross-community bridge._
- **What connects `1) docker-compose.observability.yaml`, `2) prometheus/prometheus.yml`, `3) Provisionar o datasource do Grafana (auto)` to the rest of the system?**
  _6 weakly-connected nodes found - possible documentation gaps or missing edges._