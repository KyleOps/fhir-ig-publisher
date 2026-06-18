# Opt-in client-side source viewers (`dynamic-source-viewers`)

Render the JSON/XML/Turtle "view source" pages **client-side from the raw resource file** instead of
baking a full syntax-highlighted copy of the source into every page. Opt-in, off by default.

> Stacked on the **dynamic publish box** branch (separate PR). No code overlap — this PR only touches
> the render pipeline (`PublisherGenerator`/`PublisherIGLoader`/`PublisherFields`).

## Problem

For every resource the publisher emits `X.json.html`, `X.xml.html`, `X.ttl.html` — full pages whose
body is the resource serialized **and** syntax-highlighted **and** wrapped in page chrome. That body is
the same data as the raw `X.json`/`X.xml`/`X.ttl` (which are also published), just highlighted. It is
the single biggest per-resource cost: on AU Base 6.0.0 these pages are **73.2 MB across 822 files**
(~27% of the version); large resources reach ~90 KB per viewer page.

## Change

New IG parameter **`dynamic-source-viewers`** (`true`). When set, each viewer page's body becomes a
small shell instead of the baked source:

```html
<pre class="fhir-source-dyn" data-src="StructureDefinition-au-medication.json">
  <code class="language-json">Loading source…</code></pre>
<script src="dynamic-source.js"> </script>
```

`dynamic-source.js` (one ~0.7 KB file per version, emitted once) fetches the raw file (same directory)
and highlights it with **Prism, which is already loaded on the page**. Each viewer page collapses from
up to ~90 KB to a flat ~11.8 KB (the remainder is shared page chrome). The existing tab links are
unchanged (the `X.json.html` page still exists), so no template change is needed.

Off by default — only IGs that set the parameter are affected.

### Why a `.js` file, not inline

`HTMLInspector` forbids inline `<script>` in generated pages (FATAL on the HL7 ci-build). So the JS is
shipped as a relative `dynamic-source.js` (allowed) and registered in `otherFilesRun` so it is treated
as expected output (not cleaned up). Validated: 0 inline scripts, 0 broken links.

## Trade-off

The client-side view loses the **in-source FHIR reference hyperlinks** the server-side renderer adds
(e.g. a `reference` value becoming a clickable link to the target). Syntax highlighting and the raw
content remain. Because it is opt-in, IGs that value those links simply leave it off.

## Files

- `PublisherFields` — `dynamicSourceViewers` flag.
- `PublisherIGLoader` — `case "dynamic-source-viewers"` reads the parameter.
- `PublisherGenerator` — `saveDirectResourceOutputs` emits the shell (`dynamicSourceShell`) instead of
  the baked source for `xml-html`/`json-html`/`ttl-html`; `ensureDynamicSourceJs` writes the JS file.

## Validation

Container `-ig` build of AU Base with the parameter, vs the live S3 6.0.0 (same version, baked):

| viewer pages | baked (S3 6.0.0) | A3 shells | reduction |
|---|---|---|---|
| `.json.html` / `.xml.html` / `.ttl.html` | 822 files, **73.2 MB** | 840 files, **~10.2 MB** | **~86% (~63 MB/version)** |

Build clean (0 broken links, 0 inline scripts). Each page ~90 KB → flat ~11.8 KB.

This is primarily a **size** lever — at build time it skips the server-side highlight render of
~822×3 pages/version but still writes the raw files, so the render-time saving is modest; the raw
`.json/.xml/.ttl` formats must be generated (they normally are).

## Required companion change (separate repo)

Register the `dynamic-source-viewers` code in the tooling `ig-parameters` CodeSystem/ValueSet
(`hl7.fhir.uv.tools`, `http://hl7.org/fhir/tools/CodeSystem/ig-parameters`) — as for every IG-publisher
parameter (e.g. `produce-jekyll-data`, `shownav`). Without it the IG validator reports
"code not in value set" errors for opted-in IGs. This publisher PR adds **0** new errors on its own.

## Retroactive option (no re-render)

Already-published versions can be migrated by a one-off **static rewrite** of existing viewer pages
(swap the baked `<pre>` for the shell + drop one `dynamic-source.js` per version) — the raw files
already exist, so no publisher run is needed. On AU Base that reclaims **~1.5 GB** (whole co-hosted
bucket ~3.5 GB).
