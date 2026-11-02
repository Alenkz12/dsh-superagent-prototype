# http.ts

This file is the management HTTP boundary of the orchestrator: it mounts every route the web panel and helper scripts use under `/orchestrator/api/` on the host's web server. It is the single place where an incoming loopback request is gated, matched to a route, validated, turned into a view or a runtime action, and serialized back as JSON — including the status code a caller sees when something goes wrong. Reading it tells you which operations the panel can perform on the running orchestrator, and what each one accepts and returns.

## Whole-unit functional diagram

```mermaid
flowchart LR
    panel_001["Panel or script request"] --> guard_002["Loopback and same-origin gate"]
    guard_002 --> match_003["Route matching with decoded parameters"]
    match_003 --> table_004["Route table with query and body validation"]
    table_004 --> views_005["View builders and runtime services"]
    views_005 --> result_006["JSON reply or mapped error status"]
```

A request only reaches a route after it proves it comes from the local panel on the loopback address; from there the route table decides whether the request is a read that builds a view or an action that changes orchestrator state, and the last step turns either outcome (or a failure) into a JSON body with a meaningful HTTP status.

## At a glance

- **Primary role**: API boundary — it defines the operations the panel may call, their accepted input, and the exact JSON shape and status code returned.
- **Secondary roles**: Orchestration — each route wires together a request, a validation step, and the appropriate view builder or runtime service in a fixed order.
- **Confidence**: High — the whole file was read, its registration point in <div style="display: inline; padding: 0 6px; border-radius: 4px; background-color: var(--tag-code-bg); color: var(--tag-code-fg); font-size: inherit; font-weight: 500;"><a target="_self" href="#src/app/runtime.ts:203?class=code" style="color: inherit; text-decoration: none;">runtime.ts:203</a></div> was confirmed, and its statuses and guards are exercised by <div style="display: inline; padding: 0 6px; border-radius: 4px; background-color: var(--tag-code-bg); color: var(--tag-code-fg); font-size: inherit; font-weight: 500;"><a target="_self" href="#tests/http.spec.ts?class=code" style="color: inherit; text-decoration: none;">tests/http.spec.ts</a></div>.

## What this file does

- **Mounts the whole management API in one place.** `registerHttp` registers a single prefix handler for `/orchestrator`, so the orchestrator adds one mount to the shared web server instead of dozens of individual routes.
- **Keeps the API local to the machine and to the panel.** Registration fails outright unless the web server binds `127.0.0.1`, and every request must carry `x-orchestrator-request: 1` with a loopback `Host` (and, when present, a matching `Origin`); otherwise the caller gets a 403.
- **Answers while the orchestrator is still starting up.** The handler awaits the runtime's readiness gate, so the panel can read setup problems and fix configuration before the orchestrator is fully running.
- **Exposes reads that build page views.** State, overview, search, sources, settings, threads, decisions, tickets, prompt configuration, policy history and the model catalog each delegate to a view module so the panel receives ready-to-render data.
- **Exposes operational actions.** Pausing the executor, saving settings blocks or prompts, retrying or stopping a ticket, unsubscribing a thread, requeuing or dismissing decisions, refreshing source labels, syncing policy and re-running setup all call into runtime services.
- **Normalizes input and failure.** Query strings use forgiving defaults (a bad `offset` becomes `0`, an oversized `limit` is clamped), request bodies are size-capped JSON, and every error becomes a status the panel can localize: 400 for validation, setup or route errors, 404 for missing records, 409 for revision conflicts, 500 otherwise.

## Key functions or classes

### `registerHttp(ctx, runtime)`

Registers the prefix handler, rejects non-loopback hosts, and returns a disposer so the mount disappears on shutdown.

<div style="display: flex; align-items: flex-end;">
    <span style="background-color: var(--tag-code-bg); margin: 8px 0 8px 14px; padding: 6px 12px; border-radius: 4px; border-left: 4px solid var(--tag-code-border); color: var(--tag-code-fg); font-size: 14px; font-weight: 500;">
        <a target="_self" href="#src/app/http.ts:230-247?class=code" style="color: inherit; text-decoration: none;">Code: src/app/http.ts:230-247</a>
    </span>
</div>

### `HttpHost`

The narrow runtime facade the routes depend on — store, sources, prompts, sessions, settings, engine, executor, health, deployment and setup state — which keeps the router testable without booting the real orchestrator.

<div style="display: flex; align-items: flex-end;">
    <span style="background-color: var(--tag-code-bg); margin: 8px 0 8px 14px; padding: 6px 12px; border-radius: 4px; border-left: 4px solid var(--tag-code-border); color: var(--tag-code-fg); font-size: 14px; font-weight: 500;">
        <a target="_self" href="#src/app/http.ts:31-56?class=code" style="color: inherit; text-decoration: none;">Code: src/app/http.ts:31-56</a>
    </span>
</div>

### `routes(runtime)`

Builds the route table keyed by `METHOD path`, where each handler parses what it needs and either returns a view or performs one action.

<div style="display: flex; align-items: flex-end;">
    <span style="background-color: var(--tag-code-bg); margin: 8px 0 8px 14px; padding: 6px 12px; border-radius: 4px; border-left: 4px solid var(--tag-code-border); color: var(--tag-code-fg); font-size: 14px; font-weight: 500;">
        <a target="_self" href="#src/app/http.ts:151-214?class=code" style="color: inherit; text-decoration: none;">Code: src/app/http.ts:151-214</a>
    </span>
</div>

### `match(table, method, segments)`

Finds the first route whose verb and literal path segments fit, URI-decoding each `:param` into the handler's params.

<div style="display: flex; align-items: flex-end;">
    <span style="background-color: var(--tag-code-bg); margin: 8px 0 8px 14px; padding: 6px 12px; border-radius: 4px; border-left: 4px solid var(--tag-code-border); color: var(--tag-code-fg); font-size: 14px; font-weight: 500;">
        <a target="_self" href="#src/app/http.ts:215-229?class=code" style="color: inherit; text-decoration: none;">Code: src/app/http.ts:215-229</a>
    </span>
</div>

### `failure(error)`

Chooses the status and payload for a thrown error, so callers get `issues` for schema problems, `setup` for configuration problems, `code`/`values` for user errors, and a plain `error` message for conflicts and unexpected failures.

<div style="display: flex; align-items: flex-end;">
    <span style="background-color: var(--tag-code-bg); margin: 8px 0 8px 14px; padding: 6px 12px; border-radius: 4px; border-left: 4px solid var(--tag-code-border); color: var(--tag-code-fg); font-size: 14px; font-weight: 500;">
        <a target="_self" href="#src/app/http.ts:81-91?class=code" style="color: inherit; text-decoration: none;">Code: src/app/http.ts:81-91</a>
    </span>
</div>

### `stateView(runtime)`

Assembles the `/orchestrator/api/state` payload — setup state and accounts, load figures, pause flag, master session id and per-part health — read by every panel page and by `scripts/dsh-superagent status`.

<div style="display: flex; align-items: flex-end;">
    <span style="background-color: var(--tag-code-bg); margin: 8px 0 8px 14px; padding: 6px 12px; border-radius: 4px; border-left: 4px solid var(--tag-code-border); color: var(--tag-code-fg); font-size: 14px; font-weight: 500;">
        <a target="_self" href="#src/app/http.ts:105-110?class=code" style="color: inherit; text-decoration: none;">Code: src/app/http.ts:105-110</a>
    </span>
</div>

### `readBody(req, maxBytes)`

Streams and JSON-parses a request body, refusing oversized or non-JSON payloads with a 400 so no handler receives an unvalidated blob.

<div style="display: flex; align-items: flex-end;">
    <span style="background-color: var(--tag-code-bg); margin: 8px 0 8px 14px; padding: 6px 12px; border-radius: 4px; border-left: 4px solid var(--tag-code-border); color: var(--tag-code-fg); font-size: 14px; font-weight: 500;">
        <a target="_self" href="#src/app/http.ts:62-71?class=code" style="color: inherit; text-decoration: none;">Code: src/app/http.ts:62-71</a>
    </span>
</div>

## Public contract

- **Caller**: the management panel through <div style="display: inline; padding: 0 6px; border-radius: 4px; background-color: var(--tag-code-bg); color: var(--tag-code-fg); font-size: inherit; font-weight: 500;"><a target="_self" href="#src/client/api.ts?class=code" style="color: inherit; text-decoration: none;">api.ts</a></div>, which sends the same-origin header on every call, plus operators and agents calling the documented routes directly.
- **Operation**: one JSON-over-HTTP management API at `/orchestrator/api/<route>`, where `<route>` is a name such as `state`, `tickets`, `settings/:block` or `prompts`.
- **Accepted input**: an HTTP method and path from the route table, optional query strings (paging, filters, `before`/`around`, `window=24h`), and for writes a JSON body — for example `{ "value", "revision" }` when saving a settings block, `{ "baseRevision", "config" }` for prompts, or `{ "paused": true }` for the executor.
- **Result**: a JSON payload with the view or action outcome. Failures return `{ error, … }` with `issues` for field-level validation, `setup` for configuration problems, `code`/`values` for structured user errors, and statuses 400, 403, 404, 409 or 500. The shared type contract lives in <div style="display: inline; padding: 0 6px; border-radius: 4px; background-color: var(--tag-code-bg); color: var(--tag-code-fg); font-size: inherit; font-weight: 500;"><a target="_self" href="#src/app/api.ts?class=code" style="color: inherit; text-decoration: none;">api.ts</a></div>.

### Access and validation requirements

The web server must be bound to `127.0.0.1`, and the request's `Host` must name that address or `localhost` with the server's port. Callers must send `x-orchestrator-request: 1`, and if they send an `Origin` it must match the host. Request bodies are limited to 128 KiB by default, with prompt saves and previews allowed a larger cap derived from the configured prompt size. Settings and prompt writes carry a revision, so a save based on stale data fails with 409 instead of overwriting newer changes. Because routes wait on the readiness gate, a caller never sees a half-initialized runtime; the deployment notes describe this contract in <div style="display: inline; padding: 0 6px; border-radius: 4px; background-color: var(--tag-text-bg); color: var(--tag-text-fg); font-size: inherit; font-weight: 500;"><a target="_self" href="#docs/006-deployment.md?class=text" style="color: inherit; text-decoration: none;">006-deployment.md</a></div>.

## Coordinated flow

1. The panel calls a route path under `/orchestrator/api/` with the same-origin header; the registered handler ignores anything outside that prefix.
2. The handler verifies the request is same-origin and awaits the runtime's `ready` promise, so setup-time requests are answered rather than rejected.
3. `match` selects the route, decodes path parameters, collects the query string, and hands the handler a lazily-read body reader.
4. The route parses its query (forgiving defaults, clamped limits) or body, checks that referenced records and Source blocks exist, then either builds a view from the view modules or calls the runtime service that performs the action.
5. The result is serialized as a 200 JSON reply, while any thrown error is converted by `failure` into its status and payload; a request that matches no route becomes `404 Unknown operation`.

### Important branching

Reads and writes diverge at step 4: reads only compose data, whereas writes go through the settings, prompts, engine or session services and therefore also return revision or delivery information. Two branches are worth noting for callers: saving a settings source block that names a Source that does not exist yields 404 rather than 400, and prompt saves may answer 409 when another save won the race.

## Main inputs and results

- **Inputs**: the incoming HTTP request (method, URL path, query, headers, body), the host runtime facade of type `HttpHost` supplied by the orchestrator, and the web server mount supplied by the host plugin.
- **Results**: JSON responses — page data, action confirmations such as `{ paused }`, `{ retried: true }`, `{ removed, context_delivery }`, or new revision numbers — plus a stable error status and message when the request cannot be served.
- **Visible side effects**: settings, prompts and executor pause state are changed on the orchestrator; tickets are retried or stopped; threads are unsubscribed; decisions are dismissed; a processing pass is requested; Source labels are refreshed; policy is synced; and a setup re-check schedules a restart. Panel observers see these immediately in the next poll of `GET state`.


---
name: http.ts

update-time: 2026-11-02 10:34

description: This file is the management HTTP boundary of the orchestrator: it mounts every route the web panel and helper scripts use under `/orchestrator/api/` on the host's web server. It is the single place where an incoming loopback request is gated, matched to a route, validated, turned into a view or a runtime action, and serialized back as JSON — including the status code a caller sees when something goes wrong. Reading it tells you which operations the panel can perform on the running orchestrator, and what each one accepts and returns.

---
