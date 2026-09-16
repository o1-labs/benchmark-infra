# Infrastructure for benchmarking

Docker Compose stack hosting O(1) Labs' benchmark dashboards: Traefik (TLS via Let's Encrypt
DNS-01) in front of Grafana and InfluxDB 2.x.

This repo contains **only** configuration — there is no application code and nothing here writes
benchmark data. The producers live in the `o1js` and `mina` repositories and push directly to
InfluxDB.

```
docker-compose.yaml                  # the whole stack; .env supplies every value
deployment/grafana/datasources/      # datasource.yml — read only at Grafana startup
deployment/grafana/dashboards/       # provider config + dashboard JSON — hot-reloaded
deployment/influxdb/init/            # InfluxQL 1.x leftover; unused under InfluxDB 2.x
```

## Pre-requisites

- A server with a public `IP` address and [Docker Compose](https://docs.docker.com/compose/) installed.
- Configured `DNS` records for the services listed in [docker-compose.yaml](docker-compose.yaml)
  ([.env.example](.env.example)).
  - All records should point to the same `IP` address of the server, as required by the `traefik`
    reverse proxy.

## First-time deployment

- Clone this repository and `cd` into it.
- Copy `.env.example` to `.env` and update the values.
- Run `docker compose up -d --remove-orphans` to start the services.

## Changing dashboards

Grafana's file provider polls `deployment/grafana/dashboards/` roughly every 10 seconds and reloads
changed JSON. **Editing a dashboard JSON and letting it sync is the correct way to change a
dashboard — no restart, no recreation.**

The three config paths have very different blast radii:

| What you change | How it takes effect |
|---|---|
| `dashboards/*.json` | Picked up automatically within ~10s. |
| `datasources/datasource.yml` | Read **only at Grafana startup** — needs a container restart. |
| `docker-compose.yaml`, `.env` | Changes the config hash — `docker compose up` **recreates** containers. |

> [!WARNING]
> **Recreating the Grafana container loses its database.** Compose mounts only
> `/etc/grafana/provisioning` and `/etc/localtime`, so `grafana.db` — dashboards saved through the
> UI, users, annotations, API keys — lives in the container's writable layer and is not persisted.
> `docker compose down`, `up --force-recreate`, and `docker compose pull` followed by `up` all
> discard it. `docker restart <container>` preserves it.
>
> The image tags `grafana/grafana` and `traefik:latest` are unpinned, so a `pull` can also jump
> major versions.

## Datasource UIDs are derived from the datasource name

`datasource.yml` declares no explicit `uid:`, so Grafana derives one from the name:

```
uid = "P" + upper(sha256(name).hexdigest()[:16])
```

| Name in `datasource.yml` | Derived UID |
|---|---|
| `InfluxDB (o1js)` | `PA37C35462CC6599F` |
| `InfluxDB (mina)` | `P81B2C7F097038F1D` |

UIDs are stable across rebuilds, but **renaming a datasource changes its UID and silently breaks
every dashboard that references the old one** — panels render blank with no error. This has already
broken both dashboards once. If you rename a datasource, update the dashboard JSON in the same
change:

```bash
grep -o '"uid": "[^"]*"' deployment/grafana/dashboards/*.json | sort | uniq -c
```

## Writing dashboard queries

Panels use Flux, scoped to the dashboard time picker via `v.timeRangeStart` / `v.timeRangeStop`.

Template variables interpolate as `/^${varName:regex}$/`, so **a variable that resolves to nothing
matches nothing and blanks the panel.** The most common cause is `schema.tagValues()` called
without an explicit `start:`, which defaults to a 30-day lookback and silently returns zero options
for any bucket that has not been written to recently. Always pass one:

```flux
import "influxdata/influxdb/schema"
schema.tagValues(bucket: "mina-benchmarks", tag: "gitbranch", start: v.timeRangeStart)
```

Validate JSON before opening a PR — malformed JSON is simply ignored by the provider:

```bash
python3 -m json.tool deployment/grafana/dashboards/mina-benchmarks.json > /dev/null
```

## Buckets

| Bucket | Dashboard | Notes |
|---|---|---|
| `mina-benchmarks` | `mina-benchmarks.json` | Actively written by CI. |
| `o1js-benchmarks` | `o1js-benchmarks.json` | Dormant since 2025-03-12. |
| `mina-perf-testing-benchmarks` | — | Dormant; no dashboard uses it. |

Only `o1js-benchmarks` is bootstrapped by this repo (`DOCKER_INFLUXDB_INIT_BUCKET`); the mina
buckets were created out-of-band and a clean deployment will not recreate them.

Note that two generations of producers write different tags for the same concept — `gitbranch`
(older) versus `branch` / `git_branch` (newer), and `commit` versus `git_commit`. The dashboards
filter on `gitbranch`, so they are self-consistent but only ever see part of the bucket.
