![banner2x](banner2x.png)

Möbius is an execution engine and development framework for building  auditible, re-usable and portable  analytics pipelines and applications.   

Workflows, applications and their associated runtime specifications are packaged into a single sharable artifact, designed with stability and reproducability in mind . 

When shared with another user, the recipient they author a **binding** that maps the the schema of their dataset (or sets) onto the package's expectations, and run it.  The engine absorbs the overhead costs of installation and system configuration by assembling a purpose-built virtual environment to the package's own pinned specification.  

Workflow results are emitted into a **result bundle**, a single DuckDB file with a generated [Frictionless](https://frictionlessdata.io/) `datapackage.json` containing  each result table's inferred schema and the relations between them beside it. Result bundles themselves are  valid Möbius inputs, enabling the chaining of workflows and applications. 

A package may also ship a front-end, which the engine wires to that bundle in one of two forms:

| Form                    | What it is                                                 | Query engine                                           | Use when                                                     |
| ----------------------- | ---------------------------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------ |
| **Static / standalone** | One self-contained HTML file with the result data embedded | DuckDB-WASM in the browser                             | Sharing a finished analysis — mail it, drop it in a folder, open by double-click |
| **Live**                | The same front-end served from the Möbius process          | DuckDB-WASM, plus `mobius.call()` back into the engine | Interactive work that needs computation the run did not do — extracting keywords from the cluster a user just clicked |

Both are desktop applications in the ordinary sense: local files, local compute, no cloud account, no network unless a step asks for one.

## Overview

### The Anatomy Of a Package

A package is a directory, or a `.mobius` tarball of the following spec:

```
  package/
  ├── manifest.yaml      name, version, spec version, description, author
  ├── schemas/           one Frictionless Table Schema per expected table
  ├── relations.yaml     expected relations between those tables     (optional)
  ├── pipeline.yaml      the steps, and any author-declared parameters
  ├── queries/*.sql      SQL steps, referenced by the pipeline
  ├── scripts/*.py|*.R   script steps — the core mechanism
  ├── scripts.yaml       on-demand callable scripts                  (optional)
  ├── views/             the front-end, bound to result queries      (optional)
  ├── targets.yaml       what a run produces: exports, presentation  (optional)
  ├── pyproject.toml  ┐  the pinned Python environment
  ├── uv.lock         ┘
  ├── renv.lock          the pinned R environment, if R scripts are present
  └── checksum           SHA-256 over the contents, written when sealed
```

Everything in it is plain text, so a package diffs, reviews and version-controls like any other source tree. Schemas use the Frictionless Table Schema standard, and `spec_version` in the manifest gates engine compatibility.

Packages dont contain contain any reference to a particular dataset. `schemas/` states the canonical structure of the rowsthe package processes; a binding states how one specific dataset satisfies it. That separation is what makes the same package runnable by someone whose columns are named differently.

### The Engine 

```
     CLI            hub            serve            MDM
      └──────────────┴───────┬───────┴───────────────┘
                             ▼
                          engine
                             │
   package ┐                 │
   binding ├──►  canonical tables  ──►  step DAG  ──►  result bundle
   dataset ┘         (embedded DuckDB throughout)            │
                                                             ▼
                                            manifest · exports · front-end
```

When serving or executing a package with an associated binding, the engine reshapes the bound data into the canonical structure the package expects.  No step downstream ever sees the original shape. The data processing workflow as specified by `pipeline.yaml` is then ordered into a **DAG**, inferred from what each one declares it reads and writes, and executed in dependency order.

During execution each step reads and writes named tables inside that one DuckDB instance, so the output of one step is simply an input the next can name. A `sql` step runs a query from `queries/`; `filter` and `aggregate` compile to SQL; a `script` step is an ordinary Python or R program — Möbius materialises its declared inputs as Parquet in a temporary workspace, runs it as a subprocess, and re-ingests its declared outputs as tables. A script therefore imports no SDK, no framework and nothing from Möbius: the contract is argv and files. Script steps are the only ones that execute code the package author shipped, and they refuse to run unless `--allow-scripts` was passed.

The output of the pipeline is then emitted as a **result bundle**, a run manifest, and a standalone HTML viewer (if the package ships one) to an output directory.  The entire process happens inside a embedded DuckDB instance, so there is no separate database or service to stand up once an execution is terminated, the engine holds no memory of its contents.

Users, agents and applications can interact with the engine through one of four interfaces. Every interactive action writes the same files a scripted one would: a binding authored by clicking through the browser reruns headlessly in CI.

| Interface              | Entry                              | For                                                          |
| ---------------------- | ---------------------------------- | ------------------------------------------------------------ |
| **Deployment Manager** | `mobius mdm`                       | A browser-based package management UI for **humans**         |
| **TUI**                | `mobius hub`                       | **Humans and Agents:** Binding, running and packaging in one interactive place |
| **CLI**                | `mobius run`, `bind`, `package`, … | **Humans and Agents:** Scripting, CI, headless runs          |
| **Local JSON API**     | `mobius serve`                     | **Live workflows or applications:** Other tools, and live front-ends calling back into the engine |

### Making and Using Packages 

**The Mobius engine handles all four stages of a package's lifecycle:** 

| Stage             | Description                                      | Commands                  |      |
| ----------------- | ------------------------------------------------ | ------------------------- | ---- |
| **Authoring**     | Packaging applications or workflows              | `mobius init`, `pack`     |      |
| **Binding**       | Linking a package to a dataset                   | `mobius bind`, `validate` |      |
| **Execution**     | One-time execution of standalone workflows       | `mobius run`, `view`      |      |
| OR:**Deployment** | Serving live applications or debugging workflows | `mobius mdm`, `serve`     |      |

**Authoring.** `mobius init` scaffolds the boilerplate layout for a package. You declare what the package expects of a dataset, what it does, and what it produces, write the scripts, and pin the environment. Mobius pack` then seals the result into a checksummed archive.

```
  mobius init ──► scaffolded layout
                        │
                        ▼   you declare and write
        ┌───────────────────────────────────────────────┐
        │  schemas/        what it expects of a dataset │
        │  scripts/        the work itself (Python / R) │
        │  pipeline.yaml   the steps, and what each     │
        │                  reads and writes             │
        │  targets.yaml    what a run produces          │
        │  views/          the front-end, for an app    │
        │  pyproject.toml  the pinned environment       │
        │  + uv.lock                                    │
        └───────────────────────────────────────────────┘
                        │
                        ▼   bind and run against real data until it holds
                 mobius pack ──►  name@version.mobius
                                  sealed and checksummed
```



**Binding.** The  `mobius bind` profiles your dataset (columns, types, keys, foreign-key candidates) and proposes a mapping onto the package's expected schemas. Nothing is written until you confirm it, and what gets written is split in two: the part that makes the binding correct, and the part that is only this machine's preference.

```
  your dataset ──► profile: columns · types · keys · FK candidates
                                   │
  package's expected schemas ──────┤  suggest mapping
                                   ▼
                          you confirm or edit
                                   │
             ┌─────────────────────┴──────────────────────┐
             ▼                                            ▼
       binding.yaml                          binding.yaml.run.yaml
       shareable: sources, renames,          local: interpreter, offline,
       coercions, derived expressions        parameter overrides
```



**Execution.** A run applies the binding to produce canonical tables, infers a DAG from what each step declares it reads and writes. Steps are then executed in dependency order. 

```
  binding applied ──► canonical tables ──► DAG, in dependency order
                                                  │
                    ┌─────────────────────────────┴───────────────────┐
                    │  script *   Parquet in, Parquet out             │
                    │  sql        a query from queries/               │
                    │  filter     a predicate over a table            │
                    │  aggregate  group-by into a named table         │
                    │                                                 │
                    │  * gated by --allow-scripts                     │
                    └─────────────────────────────┬───────────────────┘
                                                  ▼
                       result bundle · manifest · exports · front-end
```



**Deployment.** `mobius mdm` executes the same package and binding, but keeps what it produces. The run is given an identity, an output directory and a log of its own, recorded before the pipeline starts and updated at every state change, so a launch can be listed, stopped, and read back long after the run beneath it finished. 

Live application front-ends are served from the process itself, under `/app/<deployment-id>/` and therefore on the same origin as the JSON API. Callable scripts, defining interactive data-transformation features, are accessed from that page through `mobius.call()`. The call posts to the API on that same origin, and the engine runs the named script against the deployment's own result tables, in a workspace of its own, streaming progress back as it works; the tables it produces are returned to the page while leaving the source data untouched.

```
  package + binding + type
        │
        ▼
   queued ──► running ──┬──► serving      live: retained until stopped ──► stopped
                        ├──► ready        standalone / data: artifacts on disk
                        ├──► failed       the run returned an error
                        └──► interrupted  the process died mid-run
```


### State and Durability

| What                      | Where it lives                                           | Survives a restart |
| ------------------------- | -------------------------------------------------------- | ------------------ |
| API run and call records  | In-Memory                                                | **No**             |
| Package                   | Its own directory, or a `.mobius` archive                | Yes — shareable    |
| Binding                   | `binding.yaml`within a dataset directory                 | Yes — shareable    |
| Run output                | The `-o` directory: bundle, manifest, exports, front-end | Yes                |
| Library and index         | `<user-config-dir>/mobius/library.json`, `index.json`    | Yes — caches only  |
| Deployment record and log | `<user-config-dir>/mobius/deployments/<id>/`             | Yes                |
| Deployment output         | `<workspace>/.mobius/deployments/<id>/`                  | Yes                |

Möbius runs no daemon. A deployment outlives the session because its record is on disk. When serving live applications, the engine forgets its run and call records on restart while preserving pre-declared output bundles. 



# Quickstart 1 — Setting up Möbius

### Install a pre-built release (MacOS only)

Releases ship one tarball per platform plus a `SHA256SUMS` file. Each archive extracts to a single directory holding the `mobius` binary, `LICENSE`,`README.md`, and `USAGE.md`.

```bash
# From the repository's Releases page, download the archive and SHA256SUMS.
# Apple Silicon: _darwin_arm64   Intel: _darwin_amd64
VER=v0.1.0-Alpha
ARCHIVE=mobius_${VER}_darwin_arm64.tar.gz

shasum -a 256 --ignore-missing -c SHA256SUMS    # verify before extracting
tar -xzf "$ARCHIVE"

mkdir -p ~/.local/bin
install -m 0755 "mobius_${VER}_darwin_arm64/mobius" ~/.local/bin/mobius

mobius --help
```

If `mobius --help` is not found, `~/.local/bin` is not on your `PATH` — add it:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

The binary is unsigned and unnotarised, so macOS quarantines it when the archive
arrives via a browser. Either download with `curl -L`, or clear the flag:

```bash
xattr -d com.apple.quarantine ~/.local/bin/mobius    # if macOS refuses to run it
```

### Point Möbius at your work

Möbius holds no registry. Two indexes make packages findable, and both are caches over files you already have:

| Index      | What it holds                                                | Location                                | Override         |
| ---------- | ------------------------------------------------------------ | --------------------------------------- | ---------------- |
| Library    | Packages this instance has created or opened                 | `<user-config-dir>/mobius/library.json` | `MOBIUS_LIBRARY` |
| Scan index | Packages and bindings found under configured roots (the browser gallery) | `<user-config-dir>/mobius/index.json`   | `MOBIUS_INDEX`   |

The library starts empty and fills as you use it — any command naming a package (`inspect`, `bind`, `run`, `pack`, `init`, `package`) records it. To seed it in bulk:

```bash
mobius hub ~/analysis      # imports every package found under ~/analysis, then opens the hub
```

For the browser surface, add scan roots in Settings → scan roots. Roots must sit inside the workspace, so start `mobius mdm` at the directory that contains everything you want indexed.

### Set up a Python environment once per binding

Most analytical packages need a real interpreter. Rather than passing `--python` on every run, set it once in the hub's Run form or the Deployment Manager's launcher and tick *remember these settings* / *save as defaults*. 

That writes `binding.yaml.run.yaml` beside the binding, future runs from any surface  pre-fills from it. `binding.yaml` itself is untouched.

### Verify the install

Any install:

```bash
mobius --help
mobius init /tmp/checkpkg        # scaffold a blank package
mobius inspect /tmp/checkpkg     # read it back
```

From a source checkout, against the bundled fixtures:

```bash
mobius inspect testdata/embedder-pkg                                  # reads a package
mobius bind testdata/embedder-pkg testdata/embedder-data -y -o /tmp/b.yaml
mobius validate testdata/embedder-pkg -b /tmp/b.yaml                  # no execution
go test ./...                                                          # fast suite
```

`testdata/` is not shipped in the release archives, so the second block needs the repository. On a machine low on disk add `-p 1` to `go test`: each DuckDB-linked test binary is large and parallel linking can exhaust the volume.

---

# Quickstart 2 — Running, viewing and serving packages and bundles

### Which command

| Command               | Produces                                         | Executes the pipeline? | Use when                                                   |
| --------------------- | ------------------------------------------------ | ---------------------- | ---------------------------------------------------------- |
| `mobius validate`     | A pass/fail report and the planned step order    | No                     | Checking a binding before committing to a run              |
| `mobius run`          | Result bundle, exports, run manifest, app folder | Yes                    | Any headless or scripted execution                         |
| `mobius run --inline` | One self-contained `.html`                       | Yes                    | Producing a file to send someone                           |
| `mobius view`         | A localhost server over an existing output dir   | No                     | Opening an app you already ran                             |
| `mobius hub`          | The whole loop in a terminal UI                  | Yes                    | Interactive work in one session                            |
| `mobius serve`        | A local JSON API                                 | Yes                    | Driving Möbius from scripts or other tools                 |
| `mobius mdm`          | The browser Deployment Manager                   | Yes                    | Managing many packages, bindings, and long-lived instances |

### Run a package

```bash
mobius run <pkg> -b <binding.yaml> -o <out-dir> --allow-scripts [flags]
```

| Flag                 | Purpose                                                      |
| -------------------- | ------------------------------------------------------------ |
| `-b, --binding`      | Binding file (required)                                      |
| `-o, --out`          | Output directory (required)                                  |
| `--allow-scripts`    | Permit script steps to execute — the trust gate              |
| `--python PATH`      | Run Python steps with this interpreter instead of assembling a uv env |
| `--param name=value` | Override a declared pipeline param (repeatable)              |
| `--inline`           | Emit each app view as one self-contained `.html`             |
| `--vendor-web`       | Bundle DuckDB-WASM (~73 MB) so the app works with no network |
| `--offline`          | Assemble the Python env offline, using a bundled wheelhouse if present |
| `--cache DIR`        | Persistent model cache (`HF_HOME`) for steps declaring `network: true` |
| `--verify`           | Check the package checksum before running                    |
| `--json`             | Machine-readable result                                      |

A run writes:

| Output                             | Always?                        | Contents                                                     |
| ---------------------------------- | ------------------------------ | ------------------------------------------------------------ |
| `bundle.duckdb`                    | Yes                            | The pipeline's named result tables                           |
| `datapackage.json`                 | Yes                            | Frictionless schemas and relations for those tables          |
| `run-manifest.json`                | Yes                            | Checksums, row counts, `python_env`, `self_contained`, `used_network` |
| `*.csv` / `*.parquet`              | If declared                    | Exports named in `targets.yaml`                              |
| `app/<view>/` or `app/<view>.html` | If the package declares an app | The interactive front-end                                    |

### View the result

```bash
mobius view /tmp/out            # serves localhost and opens the browser
mobius view /tmp/out --port 8080 --no-open
```

`view` is required for a served app view, not a convenience: DuckDB-WASM uses a Web Worker and `fetch`, both blocked over `file://`, so the data layer only boots in an HTTP context. The one exception is `run --inline`, whose single HTML file opens by double-click (the WASM engine still loads from a CDN unless the run used `--vendor-web`).

`view` is a static file server. It cannot run package code, so `mobius.call()` is unavailable and `mobius.serve` is `null` — gate any UI on that rather than letting a button fail.

Möbius opens the browser with `open` on macOS and `xdg-open` on Linux. On a headless or minimal Linux host where `xdg-open` is absent, pass `--no-open` (or`--no-open --json` for `serve`/`mdm`) and follow the printed URL yourself.

### Serve the engine

```bash
mobius serve                                   # loopback, random port, random token
mobius serve --workspace ~/data --port 8080    # confine paths, pin the port
mobius serve --allow-scripts                   # permit runs to execute code
mobius serve --json                            # emit {addr,url,token,workspace}
```

The same engine over HTTP: profile a dataset, author a binding, trigger a run, follow its progress over SSE, and read result tables as Arrow, Parquet, JSON, or CSV. Use it when another program needs to drive Möbius, or when an app view needs callable scripts behind it.

`serve` holds the terminal, so background it and read its startup JSON from a file rather than a pipe:

```bash
mobius serve --workspace "$PWD" --allow-scripts --json > /tmp/mobius.json &
until [ -s /tmp/mobius.json ]; do sleep 0.2; done
A=$(python3 -c 'import json;print(json.load(open("/tmp/mobius.json"))["addr"])')
T=$(python3 -c 'import json;print(json.load(open("/tmp/mobius.json"))["token"])')

curl -H "Authorization: Bearer $T" "http://$A/api/health"
curl -N "http://$A/api/runs/$RID/events?token=$T"    # live step progress (SSE takes ?token=)
```

Limits are part of the contract: loopback only, per-session bearer token on every `/api/` route, paths confined to `--workspace`, at most four concurrent runs, and `allow_scripts` refused unless the server itself was started with`--allow-scripts`. Runs are in-memory (the last 50) and forgotten on restart.

### Manage instances

```bash
mobius mdm                     # index the cwd, open the browser
mobius mdm ~/analysis --allow-scripts
mobius serve --ui              # the same UI, without opening a browser
```

The Deployment Manager is for *management*, not authoring: a searchable gallery of every indexed package, what each can be launched as, and what is running. Its unit is a **deployment** — one launch with an identity outliving the run behind it, its own output directory, a persisted record, and a log file that survives a restart.

| Deployment type | Produces                                   | Callable scripts | Persists      |
| --------------- | ------------------------------------------ | ---------------- | ------------- |
| `live`          | The app view served from this process      | Yes              | Until stopped |
| `standalone`    | One self-contained HTML file               | No               | As a file     |
| `data`          | Result bundle and exports, no presentation | No               | As files      |

`report` is shown as unavailable — the renderer builds app views only.

### Choosing a shape

| Goal                                 | Do this                                                      |
| ------------------------------------ | ------------------------------------------------------------ |
| Send someone a finished analysis     | `mobius run --inline`, share the one HTML file               |
| Look at your own run                 | `mobius run`, then `mobius view <out-dir>`                   |
| An app whose buttons do real work    | `mobius serve --allow-scripts`, or an MDM `live` deployment  |
| Feed the result into another package | Bind `bundle.duckdb` as the next package's input             |
| Data only, no front-end              | MDM `data` type, or a package whose `targets.yaml` sets `presentation: none` |
| Work offline                         | `run --vendor-web` for the app; a vendored package for Python |

---

# Quickstart 3 — Authoring packages

### 1. Create a package template 

```bash
mobius init ~/packages/incident-report
cd ~/packages/incident-report
```

| File / directory            | Holds                                                        |
| --------------------------- | ------------------------------------------------------------ |
| `manifest.yaml`             | Name, version, spec version, description, author             |
| `schemas/*.json`            | One Frictionless Table Schema per expected input table       |
| `relations.yaml`            | Expected relations between those tables                      |
| `pipeline.yaml`             | Params and the ordered steps                                 |
| `queries/*.sql`             | SQL referenced by `sql` steps                                |
| `scripts/*.py`, `*.R`       | The script steps themselves                                  |
| `views/<name>/`             | The front-end: `view.yaml` + `index.html` + assets           |
| `targets.yaml`              | What the run emits: exports and presentation                 |
| `scripts.yaml`              | Optional on-demand callable scripts (scaffolded commented out) |
| `pyproject.toml`, `uv.lock` | The pinned Python environment                                |

### 2. Declare the dataset schema

Write one schema per input table. This is the contract every binding maps onto, so name columns for the *concept*, not for whichever dataset you happen to have.

```json
// schemas/documents.json
{
  "name": "documents",
  "fields": [
    { "name": "text_primary",   "type": "string", "title": "Primary text",
      "constraints": { "required": true } },
    { "name": "text_secondary", "type": "string", "title": "Secondary text" }
  ]
}
```

Mark only required fields as `required`. A required field that no bound dataset supplies is a hard error; an optional one that is missing is not.

For multiple tables, declare their links in `relations.yaml`:

```yaml
relations:
  - from: events.actor_id
    to: actors.id
    type: many_to_one
    required: false
```

### 3. Write the scripts

A script step is an ordinary program. Möbius materialises each declared input as Parquet in a temp workspace and re-ingests each declared output. Nothing isimported from Möbius; the contract is argv and files.

```python
# scripts/embed_text.py
import argparse, pandas as pd

p = argparse.ArgumentParser()
p.add_argument("--input", required=True)     # a Parquet file Möbius wrote
p.add_argument("--output", required=True)    # a path Möbius will read back
p.add_argument("--min-cluster-size", type=int, default=4)
a = p.parse_args()

df = pd.read_parquet(a.input)                # canonical column names, always
...
out.to_parquet(a.output + "_clusters.parquet")
```

Scripts should be written against the canonical schema declared earlier.

### 4. Declare the pipeline

Steps declare what they read (`inputs:`) and what they write (`outputs:` /`into:`); the DAG is inferred from that, not parsed out of your SQL.

```yaml
# pipeline.yaml
params:
  - name: min_cluster_size
    type: integer          # string | integer | number | boolean
    default: 4
    description: "HDBSCAN minimum cluster size; lower it for smaller datasets"

steps:
  - id: embed_cluster
    inputs: [documents]
    script:
      lang: python                    # python | r
      file: scripts/embed_text.py
      argv:
        - "--input"
        - "{in.documents}"
        - "--min-cluster-size"
        - "{param.min_cluster_size}"
        - "--output"
        - "{workspace}/out"
      outputs:
        clusters: "{workspace}/out_clusters.parquet"

  - id: top_clusters
    inputs: [clusters]
    aggregate:
      from: clusters
      group_by: [cluster_eom]
      select: ["count(*) AS n"]
      into: cluster_summary
```

Four step kinds are available; exactly one per step:

| Kind        | Does                                      | Declares its output as                                       |
| ----------- | ----------------------------------------- | ------------------------------------------------------------ |
| `script`    | Runs a bundled Python/R program           | `outputs:` (table → file), `artifacts:` (HTML/PNG collected as presentation) |
| `sql`       | Runs a query from `queries/`              | `into:`                                                      |
| `filter`    | Keeps rows matching a predicate           | `into:`                                                      |
| `aggregate` | Groups and computes aggregate expressions | `into:`                                                      |

Placeholders resolved at run time:

| Placeholder      | Resolves to                                          |
| ---------------- | ---------------------------------------------------- |
| `{in.<table>}`   | The Parquet path of a materialised input             |
| `{workspace}`    | The step's temp workspace directory                  |
| `{meta}`         | The binding's passthrough meta columns, comma-joined |
| `{param.<name>}` | A declared param's resolved value                    |

Script outputs are re-ingested by extension — `.parquet`, `.csv`, or `.json`. A step writing `.html` or `.png` under `artifacts:` becomes a visualization step whose output is collected as the run's presentation; that is how a chart-drawing script regenerates a fresh visual for each new dataset, in any language.

Use `after: [step-id]` when two steps hand off by file in the shared workspace rather than by table, so ordering cannot be inferred.

**Params.** In `argv` and `env`, `{param.x}` is substituted textually, like`{in.…}` and `{workspace}`. In a SQL context — a filter's `where`, an aggregate's`select`/`group_by` — it is substituted as a complete typed *value*: a `string`param arrives already quoted. Write `where: "tag = {param.tag}"`, never`where: "tag = '{param.tag}'"`, and compose with concatenation:

```yaml
where: "name LIKE '%' || {param.q} || '%'"     # not '%{param.q}%'
```

A param written inside a literal you opened yourself is rejected at load time. The rule is what keeps a param value from being able to rewrite the statement it sits in.

### 5. Declare the outputs

```yaml
# targets.yaml
bundle:
  exports:
    - tables: [clusters, cluster_summary]
      format: csv          # csv | json | parquet
presentation:
  kind: app                # app | report | none — the renderer builds app only
  views: [clustermap]
```

The result bundle is written whether or not anything is declared here; exports are projections of it, and the presentation is optional.

### 6. Build the front-end (for an `app`)

A view is your own HTML/JS/CSS. At run time the engine writes the result tables as Parquet beside them, copies your assets verbatim, and injects a runtime that boots DuckDB-WASM and exposes a `mobius` object. There are no engine-side component templates.

```yaml
# views/clustermap/view.yaml
title: Cluster map
entry: index.html
tables: [coords2d, clusters]     # result tables this view queries
```

```js
await mobius.ready;
const rows = await mobius.query(
  "SELECT x, y, cluster_eom FROM coords2d JOIN clusters USING (url)");
```

Because the query runs client-side, filters and interactions cost nothing afterload — repeated queries of the same shape are effectively free. Reserve`mobius.call()` for work the browser genuinely cannot do.

### 7. Optional — expose callable scripts

A pipeline step runs on every run. Work that should happen only when a user asksfor it belongs in `scripts.yaml` instead: a script invoked against a *finished* run, taking its result tables as inputs and returning detached tables.

```yaml
scripts:
  - name: keywords
    description: Extract keywords from one cluster.
    lang: python
    file: scripts/extract_keywords.py
    inputs: [clusters]                              # RESULT tables of the run
    params:
      - { name: cluster, type: integer, default: "0" }
    argv: ["--in={in.clusters}", "--cluster={param.cluster}", "--out={workspace}/nodes.parquet"]
    outputs: { nodes: "{workspace}/nodes.parquet" }
```

```js
const { tables } = await mobius.call("keywords", { cluster: 3 },
  { onProgress: e => status(e.line || e.id) });
```

A call writes nothing into the run's bundle, so two simultaneous clicks aresafe. Callables need an engine behind the page: they work under `mobius serve` and in a `live` deployment, and not in a standalone HTML file. Gate the UI on `mobius.serve` being non-null. `--allow-scripts` gates them exactly as it gates
pipeline steps.

### 8. Pin the environment

```bash
uv lock          # in the package directory: writes uv.lock from pyproject.toml
```

| `env_mode` | What makes it that                                | Ships           | Self-contained                |
| ---------- | ------------------------------------------------- | --------------- | ----------------------------- |
| `vendored` | `vendor/python/wheels/` exists                    | The wheels      | Yes                           |
| `lean`     | `uv.lock` or `pyproject.toml` at the package root | The recipe only | No — resolves on run          |
| `none`     | Neither                                           | Nothing         | No — the host supplies Python |

`lean` is the sensible default. Note that `pyproject.toml` without a `uv.lock` reads as `lean` but fails at run time, because `uv run --locked` needs the lock. R steps always use the host `Rscript` and are unaffected by any of this.

### 9. Test the package against real data

```bash
mobius inspect .                                            # confirm what you declared
mobius bind . ~/data/other-export -y -o /tmp/b.yaml         # accept the suggested mapping
mobius validate . -b /tmp/b.yaml                            
mobius run . -b /tmp/b.yaml -o /tmp/out --allow-scripts --python "$VENV"
mobius view /tmp/out
```

`validate` catches missing sources, unmapped required columns, coercion failures, and step-ordering problems before anything executes.

### 10. Seal it

```bash
mobius pack . -o /tmp/incident-report.mobius                    # lean
mobius pack . -o /tmp/incident-report.mobius --vendor-python    # bundles the wheelhouse
mobius inspect /tmp/incident-report.mobius                      # works on the archive
```

`--vendor-python` downloads the locked dependencies into the package so it assembles offline. It needs a `uv.lock` and fetches everything once.

### Authoring checklist

- [ ] Schemas name concepts, not one dataset's columns
- [ ] Only genuinely required fields are marked `required`
- [ ] Every step declares its `inputs:` and its outputs
- [ ] Anything a user might reasonably want to change is a declared param, not a constant in argv
- [ ] SQL-context params are unquoted; string params arrive quoted
- [ ] The view degrades gracefully when `mobius.serve` is `null`
- [ ] `uv.lock` exists if `pyproject.toml` does
- [ ] The package binds and runs against a dataset you did not develop against

---

## Command reference

| Command                                  | Purpose                                                      |
| ---------------------------------------- | ------------------------------------------------------------ |
| `mobius init <dir>`                      | Scaffold a blank package layout                              |
| `mobius inspect <pkg>`                   | Show manifest, env mode, schemas, pipeline, params, outputs  |
| `mobius bind <pkg> <data-dir>`           | Profile a dataset and write `binding.yaml` (TUI; `-y`, `--stub`, `--json`) |
| `mobius validate <pkg> -b <binding>`     | Check a binding and plan the steps without executing         |
| `mobius run <pkg> -b <binding> -o <dir>` | Execute; writes bundle, exports, app, run manifest           |
| `mobius view <out-dir>`                  | Serve a run's app output on localhost                        |
| `mobius pack <dir>`                      | Build a checksummed `.mobius` archive                        |
| `mobius package <workflow-dir>`          | Turn an existing script workflow into a package (`--auto` detects it) |
| `mobius hub [dir]`                       | Interactive terminal UI over all of the above (alias: `mobius -imode`) |
| `mobius serve`                           | Local JSON API (`--ui` also mounts the Deployment Manager)   |
| `mobius mdm [dir]`                       | Deployment Manager in the browser                            |
| `mobius diff`, `mobius repack`           | Not implemented yet — listed for discoverability             |

`--json` and stable exit codes are available on every command. 
