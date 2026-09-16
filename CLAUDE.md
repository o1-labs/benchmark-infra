# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Declarative config for **one** long-lived Docker Compose stack that hosts O(1) Labs' benchmark
dashboards: Traefik (TLS/ACME) → Grafana + InfluxDB 2.x.

There is no application code, no build, no test suite, and no CI. Every file is either a Compose
definition or a Grafana provisioning artifact. **Nothing in this repo writes benchmark data** —
the producers live in the `o1js` and `mina` repos and push to `https://influxdb.o1labs.org`.

```
docker-compose.yaml                      # the whole stack; .env supplies every value
deployment/grafana/datasources/          # datasource.yml  — read ONLY at Grafana startup
deployment/grafana/dashboards/           # dashboard.yml provider + 2 dashboard JSONs — hot-reloaded
deployment/influxdb/init/                # dead InfluxQL 1.x file; 2.x only runs *.sh here
```

## The one thing to know before editing: two config paths, two reload semantics

| What you edit | How it takes effect |
|---|---|
| `deployment/grafana/dashboards/*.json` | Grafana's file provider polls the directory (**~10s**, no `updateIntervalSeconds` set) and reloads changed JSON. **No restart.** |
| `deployment/grafana/datasources/datasource.yml` | Read **only at Grafana startup**. Requires a container restart. |
| `docker-compose.yaml`, `.env` | Changes the config hash → `docker compose up` **recreates** containers. |

Treat these as three separate changes with three different blast radii. Never bundle a
`datasource.yml` edit into a dashboard fix.

**Container recreation destroys Grafana's database.** Compose mounts only
`/etc/grafana/provisioning` and `/etc/localtime` into Grafana — there is **no volume for
`/var/lib/grafana`**, so `grafana.db` (UI dashboard edits, users, annotations, API keys) lives
solely in the container's writable layer. `docker compose down`, `up --force-recreate`, `pull`+`up`,
and `docker rm` all lose it permanently. `docker restart` preserves it. `grafana/grafana` and
`traefik:latest` are unpinned, so a `pull` also silently jumps major versions.

Host-specific operational detail for the live deployment lives in the untracked `CLAUDE.local.md`.

## Provisioned datasource UIDs are derived from the datasource *name*

`datasource.yml` declares no explicit `uid:`, so Grafana derives one:

```
uid = "P" + upper(sha256(name).hexdigest()[:16])
```

```bash
python3 -c 'import hashlib,sys; print("P"+hashlib.sha256(sys.argv[1].encode()).hexdigest()[:16].upper())' 'InfluxDB (mina)'
```

| Name in `datasource.yml` | Derived UID | Bucket |
|---|---|---|
| `Prometheus` | `PBFA97CFB590B2093` | — (no `prometheus` service exists; produces recurring 502s in the Grafana log) |
| `InfluxDB (o1js)` | `PA37C35462CC6599F` | `o1js-benchmarks` (`isDefault`) |
| `InfluxDB (mina)` | `P81B2C7F097038F1D` | `mina-benchmarks` |
| `InfluxDB` *(historical name)* | `P951FEA4DE68E13C5` | — removed by the rename in `60eec9d` |

UIDs are therefore **stable across container rebuilds but change if you rename a datasource**,
silently breaking every dashboard JSON that references the old one. This has already caused two
outages in this repo's history. **Renaming a datasource is a breaking change — grep the dashboard
JSONs for the old UID in the same commit.**

```bash
grep -o '"uid": "[^"]*"' deployment/grafana/dashboards/*.json | sort | uniq -c
```

Known remaining instance: `o1js-benchmarks.json` carries 6 **target-level** refs to the dead
`P951FEA4DE68E13C5` (lines 135, 448, 937, 1250, 1550, 1706). Currently inert — Grafana 10.4 resolves
queries via the **panel** datasource unless the panel is `-- Mixed --`, and none are. It breaks the
moment a panel is switched to Mixed.

A durable fix is to pin explicit `uid:` keys in `datasource.yml` matching the values above — but
that is a startup-only file, so it needs a deliberate restart, not a drive-by edit.

## Dashboards

Both are provisioned with `allowUiUpdates: true` and `editable: true`, so UI saves *can* diverge
from the files on disk. Before editing the JSON, confirm nobody has edited through the UI —
otherwise you clobber their work:

```bash
docker cp benchmark-infra-grafana-1:/var/lib/grafana/grafana.db /tmp/g.db
sqlite3 /tmp/g.db 'select external_id, check_sum, version from dashboard_provisioning;'
md5sum deployment/grafana/dashboards/*.json   # must match check_sum above (macOS: md5 -r)
```

The `version:` field inside the JSON comes from the original authoring environment and is unrelated
to the DB `version` — don't read it as evidence of anything.

| File | Grafana uid | Bucket | Shape |
|---|---|---|---|
| `mina-benchmarks.json` | `fdwhotur85f5sc` | `mina-benchmarks` | 5 overview panels + 5 collapsed rows pairwise-comparing `compatible` ⇄ `develop` |
| `o1js-benchmarks.json` | `adffpubsh4zy8e` | `o1js-benchmarks` | 3 rows (init / ECDSA / transaction), each with an `All` timeseries, a `main`-branch timeseries, and min/max gauges |

Panels are Flux over `v.timeRangeStart`/`v.timeRangeStop` with `aggregateWindow(every: v.windowPeriod)`.
Template variables interpolate as `/^${varName:regex}$/` — **an empty variable matches nothing and
blanks the panel**, which is how a broken variable query presents as "no data everywhere".

### Gotcha: `schema.tagValues()` without `start:` defaults to `-30d`

Every variable query in both dashboards omits `start:`. On a dormant bucket every dropdown returns
zero options and the whole dashboard goes blank; on a live bucket it silently truncates the
dropdowns to whatever appeared in the last 30 days. Always pass an explicit `start:` —
`v.timeRangeStart` to follow the time picker, or a fixed `-10y` for archive browsing.

### Gotcha: the Mina bucket has split tag names

Two generations of producers write different tags for the same concept: `gitbranch` (older) vs
`branch`/`git_branch` (newer), and `commit` vs `git_commit`. The dashboard consistently uses
`gitbranch`, so it is self-consistent but only ever sees a slice of the bucket. Also present as
*tag keys*: `compatible`, `develop`, `master`, `archive` — a writer bug emitting tag values in the
key position. `fieldFilterQuery` is labelled "Category" but queries `tag: "_field"`, where ~98% of
the 2,200+ keys are numeric junk from the same class of writer bug. These are upstream problems
this repo cannot fix.

## Buckets

| Bucket | Status | Bootstrapped by this repo? |
|---|---|---|
| `mina-benchmarks` | Live, actively written by CI | No — created out-of-band |
| `o1js-benchmarks` | Dormant since 2025-03-12 | Yes (`DOCKER_INFLUXDB_INIT_BUCKET`) |
| `mina-perf-testing-benchmarks` | Dormant, 100 rows, no dashboard | No |

Only `o1js-benchmarks` is reproducible from this repo. A clean re-deploy would not recreate the
mina buckets.

## Working commands

```bash
# Validate dashboard JSON before committing (the only "build check" that exists)
python3 -m json.tool deployment/grafana/dashboards/mina-benchmarks.json > /dev/null

# Read-only Flux query against the live InfluxDB (run on the deployment host)
TOKEN=$(grep -E '^INFLUXDB_TOKEN=' .env | cut -d= -f2-)
curl -s -H "Authorization: Token $TOKEN" \
  -H "Content-Type: application/vnd.flux" -H "Accept: application/csv" \
  --data-binary 'import "influxdata/influxdb/schema"
                 schema.tagValues(bucket: "mina-benchmarks", tag: "gitbranch", start: -7d)' \
  "http://localhost:8086/api/v2/query?org=o1labs"
```

Quote any URL containing `?` — this is a zsh environment with `NOMATCH` on, and an unquoted glob
character aborts the command before it runs.

## Conventions

- `.env` is gitignored and holds every secret. Secrets are also visible via `docker inspect`; never
  paste values into tracked files, dashboard JSON, or commit messages.
- `datasource.yml` interpolates `${INFLUXDB_ORG}` etc. from the **Grafana container's** environment,
  which `docker-compose.yaml` must pass through explicitly. Adding a variable to `datasource.yml`
  means adding it to the compose `environment:` block too.
- `datasource.yml`'s `database: o1labs` and `deployment/influxdb/init/influxdb-init.iql` are both
  InfluxDB 1.x leftovers, ignored under `version: Flux`. Likewise `GF_AUTH_ORG_ROLE` in compose is
  not a real Grafana setting (the real key is `GF_USERS_AUTO_ASSIGN_ORG_ROLE`).
- Dashboard changes ship via PR (see `#7`), not by editing files on the server.
