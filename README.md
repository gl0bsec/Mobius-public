![banner2x](banner2x.png)

Möbius packages a data-transformation workflow — Python/R scripts, SQL, and a front-end — into a single portable artifact, then re-runs it on *similarly but not identically structured* datasets. 

The recipient never edits the workflow to fit their data; they write a **binding** that maps their columns onto the package's expectations, and run it.

It is one static binary. No daemon, no server to stand up, no telemetry, and no dependency beyond the machine's own Python or R when a package uses them.

## What it is for

**Reusing analysis, not rewriting it.** An analytical workflow is usually
written once against one dataset and then either rewritten for the next one or
abandoned. Möbius separates the two halves: the package declares what shape of
data it needs and what it does with it; the binding says how *this* dataset
satisfies that shape. Scripts always receive canonical column names, so a
pipeline written against dataset A runs unchanged on A′ — a differently-named
export, a different month, a different source entirely.

**Turning a pipeline into an application.** Every run produces a result bundle
(a DuckDB file plus a Frictionless `datapackage.json`). A package may also ship
a front-end, which the engine wires to that bundle in one of two forms:

| Form | What it is | Query engine | Use when |
|---|---|---|---|
| **Static / standalone** | One self-contained HTML file with the result data embedded | DuckDB-WASM in the browser | Sharing a finished analysis — mail it, drop it in a folder, open by double-click |
| **Live** | The same front-end served from the Möbius process | DuckDB-WASM, plus `mobius.call()` back into the engine | Interactive work that needs computation the run did not do — extracting keywords from the cluster a user just clicked |

Both are desktop applications in the ordinary sense: local files, local
compute, no cloud account, no network unless a step asks for one.

**Composition.** A result bundle is itself a valid Möbius input, so one
package's output binds directly as another's input.

## Core concepts

| Concept | Definition |
|---|---|
| **Bundle** | The unit of data: a named set of tables (CSV/Parquet/JSON/SQLite/DuckDB) plus declared relations between them |
| **Package** | A versioned, shareable artifact: expected schemas + relations, pipeline steps, views, metadata, checksum |
| **Binding** | A local, per-dataset adapter: table sources, column renames, type coercions, derived expressions, relation overrides. The package is never modified |
| **Run** | Execution of a package against a bundle via a binding, producing a result bundle, optional exports, an optional application, and a run manifest |

## The architecture, and why it matters

**One engine, four surfaces.** The CLI, the terminal hub, the local JSON API,
and the browser Deployment Manager are all clients of the same engine. Nothing
is reachable from one surface and not the others; every interactive action
writes the same files a scripted one would. A binding built by clicking through
the browser binder is byte-identical in kind to one written by hand, and reruns
headlessly in CI.

**Everything is plain text and plain files.** A package is a directory (or a
tarball of one) of YAML, JSON, SQL, and scripts. It is diffable, reviewable, and
version-controllable. There is no registry and no database: packages are shared
however you share any other file, and Möbius's own index is a cache over the
filesystem, not a source of truth.

**Embedded DuckDB as the substrate.** The bundle is loaded as tables; steps read
and write named tables within one embedded DuckDB instance. Script inputs are
materialised as Parquet and declared outputs re-ingested, so a step is an
ordinary program reading and writing files — no SDK, no framework, no import of
Möbius itself. Steps form a DAG inferred from what each declares it reads and
writes.

**The binding layer is the load-bearing idea.** Because adaptation lives in one
declarative file outside the package, the package stays immutable and its
provenance stays intact. Ten analysts with ten differently-shaped exports share
one workflow and ten small YAML files, rather than ten forks that drift.

**Separation of what is shareable from what is local.** `binding.yaml` holds
what makes a binding correct. Machine-local choices — which Python interpreter,
offline or not, parameter overrides — live in a sidecar (`binding.yaml.run.yaml`)
that never travels with the shared file.

**Execution is gated, not sandboxed.** Packages carry code; `inspect` lists
every script a package would execute, and `--allow-scripts` is the explicit
opt-in to run them. The HTTP API refuses to let a client enable execution the
operator did not enable at startup, binds loopback only, requires a per-session
token, and confines caller-supplied paths to a workspace root.

**Reproducibility is recorded.** Every run writes a manifest: package checksum,
binding hash, engine version, per-step row counts, which Python environment was
resolved, and whether the run was self-contained.

---

# Quickstart 1 — Setting up Möbius

### Install a pre-built release (MacOS only)

Releases ship one tarball per platform plus a `SHA256SUMS` file. Each archive
extracts to a single directory holding the `mobius` binary, `LICENSE`,
`README.md`, and `USAGE.md`.

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

Möbius holds no registry. Two indexes make packages findable, and both are
caches over files you already have:

| Index | What it holds | Location | Override |
|---|---|---|---|
| Library | Packages this instance has created or opened | `<user-config-dir>/mobius/library.json` | `MOBIUS_LIBRARY` |
| Scan index | Packages and bindings found under configured roots (the browser gallery) | `<user-config-dir>/mobius/index.json` | `MOBIUS_INDEX` |

The library starts empty and fills as you use it — any command naming a package
(`inspect`, `bind`, `run`, `pack`, `init`, `package`) records it. To seed it in
bulk:

```bash
mobius hub ~/analysis      # imports every package found under ~/analysis, then opens the hub
```

For the browser surface, add scan roots in Settings → scan roots. Roots must sit
inside the workspace, so start `mobius mdm` at the directory that contains
everything you want indexed.

### Set up a Python environment once per binding

Most analytical packages need a real interpreter. Rather than passing `--python`
on every run, set it once in the hub's Run form or the Deployment Manager's
launcher and tick *remember these settings* / *save as defaults*. That writes
`binding.yaml.run.yaml` beside the binding, and every later run — from any
surface — pre-fills from it. `binding.yaml` itself is untouched.

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

`testdata/` is not shipped in the release archives, so the second block needs the
repository. On a machine low on disk add `-p 1` to `go test`: each DuckDB-linked
test binary is large and parallel linking can exhaust the volume.

---

# Quickstart 2 — Running, viewing and serving packages and bundles

### Which command

| Command | Produces | Executes the pipeline? | Use when |
|---|---|---|---|
| `mobius validate` | A pass/fail report and the planned step order | No | Checking a binding before committing to a run |
| `mobius run` | Result bundle, exports, run manifest, app folder | Yes | Any headless or scripted execution |
| `mobius run --inline` | One self-contained `.html` | Yes | Producing a file to send someone |
| `mobius view` | A localhost server over an existing output dir | No | Opening an app you already ran |
| `mobius hub` | The whole loop in a terminal UI | Yes | Interactive work in one session |
| `mobius serve` | A local JSON API | Yes | Driving Möbius from scripts or other tools |
| `mobius mdm` | The browser Deployment Manager | Yes | Managing many packages, bindings, and long-lived instances |

### Run a package

```bash
mobius run <pkg> -b <binding.yaml> -o <out-dir> --allow-scripts [flags]
```

| Flag | Purpose |
|---|---|
| `-b, --binding` | Binding file (required) |
| `-o, --out` | Output directory (required) |
| `--allow-scripts` | Permit script steps to execute — the trust gate |
| `--python PATH` | Run Python steps with this interpreter instead of assembling a uv env |
| `--param name=value` | Override a declared pipeline param (repeatable) |
| `--inline` | Emit each app view as one self-contained `.html` |
| `--vendor-web` | Bundle DuckDB-WASM (~73 MB) so the app works with no network |
| `--offline` | Assemble the Python env offline, using a bundled wheelhouse if present |
| `--cache DIR` | Persistent model cache (`HF_HOME`) for steps declaring `network: true` |
| `--verify` | Check the package checksum before running |
| `--json` | Machine-readable result |

A run writes:

| Output | Always? | Contents |
|---|---|---|
| `bundle.duckdb` | Yes | The pipeline's named result tables |
| `datapackage.json` | Yes | Frictionless schemas and relations for those tables |
| `run-manifest.json` | Yes | Checksums, row counts, `python_env`, `self_contained`, `used_network` |
| `*.csv` / `*.parquet` | If declared | Exports named in `targets.yaml` |
| `app/<view>/` or `app/<view>.html` | If the package declares an app | The interactive front-end |

### View the result

```bash
mobius view /tmp/out            # serves localhost and opens the browser
mobius view /tmp/out --port 8080 --no-open
```

`view` is required for a served app view, not a convenience: DuckDB-WASM uses a
Web Worker and `fetch`, both blocked over `file://`, so the data layer only boots
in an HTTP context. The one exception is `run --inline`, whose single HTML file
opens by double-click (the WASM engine still loads from a CDN unless the run
used `--vendor-web`).

`view` is a static file server. It cannot run package code, so `mobius.call()`
is unavailable and `mobius.serve` is `null` — gate any UI on that rather than
letting a button fail.

Möbius opens the browser with `open` on macOS and `xdg-open` on Linux. On a
headless or minimal Linux host where `xdg-open` is absent, pass `--no-open` (or
`--no-open --json` for `serve`/`mdm`) and follow the printed URL yourself.

### Serve the engine

```bash
mobius serve                                   # loopback, random port, random token
mobius serve --workspace ~/data --port 8080    # confine paths, pin the port
mobius serve --allow-scripts                   # permit runs to execute code
mobius serve --json                            # emit {addr,url,token,workspace}
```

The same engine over HTTP: profile a dataset, author a binding, trigger a run,
follow its progress over SSE, and read result tables as Arrow, Parquet, JSON, or
CSV. Use it when another program needs to drive Möbius, or when an app view
needs callable scripts behind it.

`serve` holds the terminal, so background it and read its startup JSON from a
file rather than a pipe:

```bash
mobius serve --workspace "$PWD" --allow-scripts --json > /tmp/mobius.json &
until [ -s /tmp/mobius.json ]; do sleep 0.2; done
A=$(python3 -c 'import json;print(json.load(open("/tmp/mobius.json"))["addr"])')
T=$(python3 -c 'import json;print(json.load(open("/tmp/mobius.json"))["token"])')

curl -H "Authorization: Bearer $T" "http://$A/api/health"
curl -N "http://$A/api/runs/$RID/events?token=$T"    # live step progress (SSE takes ?token=)
```

Limits are part of the contract: loopback only, per-session bearer token on
every `/api/` route, paths confined to `--workspace`, at most four concurrent
runs, and `allow_scripts` refused unless the server itself was started with
`--allow-scripts`. Runs are in-memory (the last 50) and forgotten on restart.

### Manage instances

```bash
mobius mdm                     # index the cwd, open the browser
mobius mdm ~/analysis --allow-scripts
mobius serve --ui              # the same UI, without opening a browser
```

The Deployment Manager is for *management*, not authoring: a searchable gallery
of every indexed package, what each can be launched as, and what is running. Its
unit is a **deployment** — one launch with an identity outliving the run behind
it, its own output directory, a persisted record, and a log file that survives a
restart.

| Deployment type | Produces | Callable scripts | Persists |
|---|---|---|---|
| `live` | The app view served from this process | Yes | Until stopped |
| `standalone` | One self-contained HTML file | No | As a file |
| `data` | Result bundle and exports, no presentation | No | As files |

`report` is shown as unavailable — the renderer builds app views only.

### Choosing a shape

| Goal | Do this |
|---|---|
| Send someone a finished analysis | `mobius run --inline`, share the one HTML file |
| Look at your own run | `mobius run`, then `mobius view <out-dir>` |
| An app whose buttons do real work | `mobius serve --allow-scripts`, or an MDM `live` deployment |
| Feed the result into another package | Bind `bundle.duckdb` as the next package's input |
| Data only, no front-end | MDM `data` type, or a package whose `targets.yaml` sets `presentation: none` |
| Work offline | `run --vendor-web` for the app; a vendored package for Python |

---

# Quickstart 3 — Authoring packages

This is the hand-authoring path: you write the package layout yourself. It is
the most explicit route and the one that gives full control over schemas,
parameters, and views.

> Two other routes exist and produce the same layout. `mobius package --reference
> <data> --step "<cmd>"` traces an existing script chain once and derives the
> declarations from what it observed. `mobius package --auto` additionally
> detects the scripts, interpreter, reference data, and I/O flags for you. Use
> the manual path below when you want to decide the contract rather than have it
> inferred.

### 1. Scaffold

```bash
mobius init ~/packages/incident-report
cd ~/packages/incident-report
```

| File / directory | Holds |
|---|---|
| `manifest.yaml` | Name, version, spec version, description, author |
| `schemas/*.json` | One Frictionless Table Schema per expected input table |
| `relations.yaml` | Expected relations between those tables |
| `pipeline.yaml` | Params and the ordered steps |
| `queries/*.sql` | SQL referenced by `sql` steps |
| `scripts/*.py`, `*.R` | The script steps themselves |
| `views/<name>/` | The front-end: `view.yaml` + `index.html` + assets |
| `targets.yaml` | What the run emits: exports and presentation |
| `scripts.yaml` | Optional on-demand callable scripts (scaffolded commented out) |
| `pyproject.toml`, `uv.lock` | The pinned Python environment |

### 2. Declare what the package expects

Write one schema per input table. This is the contract every binding maps onto,
so name columns for the *concept*, not for whichever dataset you happen to have.

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

Mark only genuinely required fields `required`. A required field that no bound
dataset supplies is a hard error; an optional one that is missing is not.

For multiple tables, declare their links in `relations.yaml`:

```yaml
relations:
  - from: events.actor_id
    to: actors.id
    type: many_to_one
    required: false
```

### 3. Write the scripts

A script step is an ordinary program. Möbius materialises each declared input as
Parquet in a temp workspace and re-ingests each declared output. Nothing is
imported from Möbius; the contract is argv and files.

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

Scripts see canonical names — the binding has already normalised the data — so
write against your own schema and never against a source dataset.

### 4. Declare the pipeline

Steps declare what they read (`inputs:`) and what they write (`outputs:` /
`into:`); the DAG is inferred from that, not parsed out of your SQL.

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

| Kind | Does | Declares its output as |
|---|---|---|
| `script` | Runs a bundled Python/R program | `outputs:` (table → file), `artifacts:` (HTML/PNG collected as presentation) |
| `sql` | Runs a query from `queries/` | `into:` |
| `filter` | Keeps rows matching a predicate | `into:` |
| `aggregate` | Groups and computes aggregate expressions | `into:` |

Placeholders resolved at run time:

| Placeholder | Resolves to |
|---|---|
| `{in.<table>}` | The Parquet path of a materialised input |
| `{workspace}` | The step's temp workspace directory |
| `{meta}` | The binding's passthrough meta columns, comma-joined |
| `{param.<name>}` | A declared param's resolved value |

Script outputs are re-ingested by extension — `.parquet`, `.csv`, or `.json`.
A step writing `.html` or `.png` under `artifacts:` becomes a visualization step
whose output is collected as the run's presentation; that is how a chart-drawing
script regenerates a fresh visual for each new dataset, in any language.

Use `after: [step-id]` when two steps hand off by file in the shared workspace
rather than by table, so ordering cannot be inferred.

**Params.** In `argv` and `env`, `{param.x}` is substituted textually, like
`{in.…}` and `{workspace}`. In a SQL context — a filter's `where`, an aggregate's
`select`/`group_by` — it is substituted as a complete typed *value*: a `string`
param arrives already quoted. Write `where: "tag = {param.tag}"`, never
`where: "tag = '{param.tag}'"`, and compose with concatenation:

```yaml
where: "name LIKE '%' || {param.q} || '%'"     # not '%{param.q}%'
```

A param written inside a literal you opened yourself is rejected at load time.
The rule is what keeps a param value from being able to rewrite the statement it
sits in.

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

The result bundle is written whether or not anything is declared here; exports
are projections of it, and the presentation is optional.

### 6. Build the front-end (for an `app`)

A view is your own HTML/JS/CSS. At run time the engine writes the result tables
as Parquet beside them, copies your assets verbatim, and injects a runtime that
boots DuckDB-WASM and exposes a `mobius` object. There are no engine-side
component templates.

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

Because the query runs client-side, filters and interactions cost nothing after
load — repeated queries of the same shape are effectively free. Reserve
`mobius.call()` for work the browser genuinely cannot do.

### 7. Optional — expose callable scripts

A pipeline step runs on every run. Work that should happen only when a user asks
for it belongs in `scripts.yaml` instead: a script invoked against a *finished*
run, taking its result tables as inputs and returning detached tables.

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

A call writes nothing into the run's bundle, so two simultaneous clicks are
safe. Callables need an engine behind the page: they work under `mobius serve`
and in a `live` deployment, and not in a standalone HTML file. Gate the UI on
`mobius.serve` being non-null. `--allow-scripts` gates them exactly as it gates
pipeline steps.

### 8. Pin the environment

```bash
uv lock          # in the package directory: writes uv.lock from pyproject.toml
```

| `env_mode` | What makes it that | Ships | Self-contained |
|---|---|---|---|
| `vendored` | `vendor/python/wheels/` exists | The wheels | Yes |
| `lean` | `uv.lock` or `pyproject.toml` at the package root | The recipe only | No — resolves on run |
| `none` | Neither | Nothing | No — the host supplies Python |

`lean` is the sensible default. Note that `pyproject.toml` without a `uv.lock`
reads as `lean` but fails at run time, because `uv run --locked` needs the lock.
R steps always use the host `Rscript` and are unaffected by any of this.

### 9. Test the package against real data

Bind it to a dataset that is *not* the one you developed against — that is the
whole point of the contract:

```bash
mobius inspect .                                            # confirm what you declared
mobius bind . ~/data/other-export -y -o /tmp/b.yaml         # accept the suggested mapping
mobius validate . -b /tmp/b.yaml                            # no execution
mobius run . -b /tmp/b.yaml -o /tmp/out --allow-scripts --python "$VENV"
mobius view /tmp/out
```

`validate` catches missing sources, unmapped required columns, coercion
failures, and step-ordering problems before anything executes.

### 10. Seal it

```bash
mobius pack . -o /tmp/incident-report.mobius                    # lean
mobius pack . -o /tmp/incident-report.mobius --vendor-python    # bundles the wheelhouse
mobius inspect /tmp/incident-report.mobius                      # works on the archive
```

`--vendor-python` downloads the locked dependencies into the package so it
assembles offline. It needs a `uv.lock` and fetches everything once.

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

| Command | Purpose |
|---|---|
| `mobius init <dir>` | Scaffold a blank package layout |
| `mobius inspect <pkg>` | Show manifest, env mode, schemas, pipeline, params, outputs |
| `mobius bind <pkg> <data-dir>` | Profile a dataset and write `binding.yaml` (TUI; `-y`, `--stub`, `--json`) |
| `mobius validate <pkg> -b <binding>` | Check a binding and plan the steps without executing |
| `mobius run <pkg> -b <binding> -o <dir>` | Execute; writes bundle, exports, app, run manifest |
| `mobius view <out-dir>` | Serve a run's app output on localhost |
| `mobius pack <dir>` | Build a checksummed `.mobius` archive |
| `mobius package <workflow-dir>` | Turn an existing script workflow into a package (`--auto` detects it) |
| `mobius hub [dir]` | Interactive terminal UI over all of the above (alias: `mobius -imode`) |
| `mobius serve` | Local JSON API (`--ui` also mounts the Deployment Manager) |
| `mobius mdm [dir]` | Deployment Manager in the browser |
| `mobius diff`, `mobius repack` | Not implemented yet — listed for discoverability |

`--json` and stable exit codes are available on every command.

## Known limits

- `report` / PDF presentations are declared by the spec and refused by the
  renderer, which builds `app` views only.
- `diff` and `repack` are stubs.
- The binder's multi-table relations editor is deferred on every surface; the
  browser binder additionally has no expression editor and writes one source per
  table. Hand-edit `binding.yaml` for either.
- Package authoring is CLI and hub only; the browser surface does not author.
- Runs held by `mobius serve` are in-memory (last 50) and forgotten on restart.
  MDM's deployments are persisted to disk and are not.
- DuckDB-WASM runs single-threaded with a ~3.1 GiB memory ceiling, so a very
  large result belongs in a `live` deployment or an export rather than an
  embedded standalone file.

## Further reading

| Document | Contents |
|---|---|
| [USAGE.md](USAGE.md) | Full command-by-command guide, runnable against `testdata/` |
| [mobius-spec.md](mobius-spec.md) | Design rationale, formats, milestones, resolved decisions |
| [COMPONENTS.md](COMPONENTS.md) | Package-by-package classification of the Go source |
| [BENCHMARKS.md](BENCHMARKS.md) | Measured performance against Streamlit and Next.js |
