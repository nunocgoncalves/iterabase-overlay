# Gateway tool bundles

`tools/` contains reviewed, operator-deployed gateway tools loaded from the exact Flux `GitRepository` artifact. These are **not pi extensions** and never execute inside AgentSandboxes.

## Layout

```text
tools/
├── manifest.schema.json
├── product/<tool>/manifest.json + index.mjs
└── client/<tool>/manifest.json + index.mjs
```

`product/` is upstream-owned. Client forks never edit it; they add tools under `client/`. One logical tool name may appear once per layer. A client tool shadows a product tool with that name, but it must publish a new version. Reusing a version with different content rejects the entire revision.

`index.mjs` must be a self-contained Node 24 ESM bundle: no runtime `npm install`, package lookup, relative/dynamic import, native addon, pi extension, or generated/customer-uploaded code. Static imports are limited to explicit `node:` built-ins. It exports:

```js
export const identity = { name: "graph.read_mail", version: "1.0.0" };
export async function invoke(context, argumentsValue) {
  return { result: { /* bounded JSON */ }, artifactRefs: [] };
}
```

The frozen invocation context contains only the declared credential slots, stable idempotency key, invocation-scoped artifact helpers, and an `AbortSignal`. Tools must not log credentials or return them in results/errors. A reviewed tool may return a safe structured failure by throwing an `Error` subclass named `ToolError` with non-empty `code`, `message`, optional boolean `retryable`, and optional JSON-safe `details` fields. Other exceptions are reduced to a non-retryable `internal` error without a stack trace or original message.

## Immutable digest

`manifest.digest` is SHA-256 over:

1. the recursively key-sorted JSON manifest projection with `digest` omitted;
2. one NUL byte;
3. the exact `index.mjs` bytes.

Use the released runner image to compute and validate the same contract used in-cluster:

```sh
docker run --rm -v "$PWD:/overlay:ro" \
  ghcr.io/nunocgoncalves/control-plane-tool-runner:0.0.19 \
  digest /overlay/tools/client/<tool>

docker run --rm -v "$PWD:/overlay:ro" \
  ghcr.io/nunocgoncalves/control-plane-tool-runner:0.0.19 \
  validate /overlay
```

Copy the digest output into `manifest.json`, then commit the bundle and manifest together. Any manifest digest or bundle-exported name/version mismatch fails before registration and leaves the last valid generation serving.

## Rollout

Flux materializes the client fork revision and digest. The runner loads a complete valid generation atomically. New attempts pin its healthy versions; existing attempts retain old pins while those versions drain. Unavailable pins fail explicitly and never substitute another version.
