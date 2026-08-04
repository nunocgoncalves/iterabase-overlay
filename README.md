# iterabase-overlay

The **product base overlay repo** for the Iterabase platform — the upstream template each client deployment forks. Part of the forge overlay system (HOR-341).

> **Status:** base prod recipe landed (HOR-299). HOR-397 adds reviewed gateway-tool bundle trees materialized from the exact Flux artifact; CRD instances remain deployment-specific.

## What this is

Iterabase is **per-customer, fully isolated, self-hosted**. Each engagement installs a complete platform stack on the customer's infra. The deployment-specific configuration — Helm values + CRD instances — lives in an **overlay repo** that `forge apply --overlay` consumes. This repo (`iterabase-overlay`) is the **upstream template** every client overlay forks.

The fork model (Horizonshift Platform Direction §5):

- **Base repo (this repo)** = the product template: base Helm values + base CRD instances, versioned with the platform.
- **Client overlay repo = a fork** of this repo. Client-specific config lives in client-owned paths (no merge conflicts on upstream sync).
- **Clients opt in to product updates** by syncing their fork with upstream (`git merge`/`rebase`). The default is tracking upstream `master`, and this deliberate sync IS the safety gate — pushes to this base repo have **no effect** on client infra until the client syncs. Pinning to a tag is optional (see below).
- **No base+client merge at apply time** — the "merge" is a git-level fork sync, client-initiated. `forge apply --overlay` takes ONE input (the client fork, self-contained).
- **Flux** (HOR-292) watches the **client fork only** → push-to-Git auto-reconciles the overlay's `crds/client/`. Base-repo pushes don't trigger reconciliation. Enabled per-deployment in `forge.yaml` (`flux.enabled: true`); Flux's wiring (GitRepository + Kustomization + token Secret) is **forge-applied to the cluster**, not repo-resident (`flux install`, not `flux bootstrap git`) — the overlay repo stays pure.

## Repo structure

```
iterabase-overlay/
├── README.md              # this file (the fork-model design doc)
├── values.yaml            # base Helm values (deployment-recipe layer over the iterabase-platform chart defaults)
├── values.client.yaml     # client Helm value overrides (empty in base; clients fill their fork)
├── crds/
│   ├── base/              # product base CRD instances
│   └── client/            # client instances + supersede patches
└── tools/
    ├── manifest.schema.json
    ├── product/           # reviewed upstream self-contained ESM bundles
    └── client/            # reviewed client bundles; logical-name precedence
```

**CRD instances, not definitions.** The CRD *definitions* (the schemas for `Model`, `ModelBackend`, `PermissionPolicy`, `IdentityMapping`) ship with the control-plane Helm chart. This overlay carries **instances** (resources of those kinds). `forge apply` installs the chart first, then `kubectl apply -k crds/client/`, so the kinds exist before instances are applied.

**Kustomize base/overlay mirrors the git fork model.** `crds/base/` is the upstream product instances (inherited via the fork); `crds/client/` composes `../base` and adds client instances + patches. This is Flux-native: HOR-292 points a Flux `Kustomization` at `crds/client/` for continuous reconciliation, with zero restructure.

## Supersede, not edit

Clients **never edit** `crds/base/` or `values.yaml` — that would cause merge conflicts on upstream sync. Instead:

- **Helm values:** put overrides in `values.client.yaml`. `forge apply` runs `helm -f values.yaml -f values.client.yaml` (later file wins).
- **CRD instances:** add client instances in `crds/client/` (client-owned filenames). To **supersede** a base instance, add a kustomize **patch** in `crds/client/kustomization.yaml` with the same identity (name); the patch overrides the base instance.

## Forking + syncing

1. **Fork** this repo to the client's Git host (e.g. `github.com/<client>/iterabase-overlay`).
2. **Add upstream** (one-time):
   ```sh
   git remote add upstream https://github.com/nunocgoncalves/iterabase-overlay.git
   ```
3. **Sync** with product updates (client-initiated; base pushes have no effect until you sync):
   ```sh
   git fetch upstream
   git merge upstream/master   # or: git rebase upstream/master
   ```
   Because client config lives in `values.client.yaml` + `crds/client/` (dedicated paths), upstream syncs rarely conflict.
4. **Pin for stability** (optional): the default is tracking upstream `master` (the deliberate sync above is the gate). To freeze on a known state, cut a tag on your fork and set `overlay.ref` to it. Base release tags are optional markers, not a workflow dependency.

## `forge apply --overlay`

```sh
forge apply --overlay https://github.com/<client>/iterabase-overlay.git
```

`forge apply` clones the client fork on the host, then:

1. `helm upgrade --install <release> <iterabase-platform-chart> -f values.yaml -f values.client.yaml`
2. `kubectl apply -k crds/client/` (after the chart, so CRD kinds exist)

`forge apply` is idempotent (re-clones to the current ref each run; `helm upgrade --install` + `kubectl apply -k` are idempotent). Private repos: `forge` prompts for a token (non-echo, scope-checked) or reads `FORGE_OVERLAY_TOKEN`.

## Gateway tools

`tools/product/` and `tools/client/` contain reviewed, self-contained Node 24 ESM gateway tools. They are loaded from the exact Flux `GitRepository` revision/digest by the trusted runner and never execute in AgentSandboxes. Client logical names supersede product names only through a new immutable version; collisions reject the revision. See [`tools/README.md`](tools/README.md) for the manifest, digest, validation, and rollout contract.

Pi skills/extensions remain a separate `pi/` tree and are not gateway tools.

## References

- Horizonshift Platform Direction §5 (overlay fork model) + §7 (tools/skills).
- forge: `forge apply --overlay` — HOR-341.
- Flux GitOps reconciliation — HOR-292.
- OPO1 client fork + full deploy — HOR-299.
- `pi/` tree + AgentSandbox harness — HOR-351.
