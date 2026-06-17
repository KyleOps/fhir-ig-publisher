# Opt-in dynamic (client-side) "current version" in the publish box

## Problem

Every published IG page carries a "publish box" banner. For non-current versions it reads, e.g.:

> This page is part of the AU Base IG (v4.0.0: R3) … **The current version which supersedes this
> version is [6.0.0](http://hl7.org.au/fhir)**. For a full list of available versions, see the
> Directory of published versions.

The current version is **hard-coded into the HTML of every page of every past version**. So cutting
a new milestone (which changes "the current version") forces the publisher to **rewrite the publish
box in every page of every prior version** — for a large multi-version IG that is tens of thousands
of files re-written and re-uploaded on every milestone. This is the single biggest cost of a
milestone release (size of the diff to sync, build time, and a long history of fragile milestone
runs).

## Change

Add an **opt-in** mode (`publish-setup.json` → `website.dynamic-publish-box: true`) that emits the
"current version" reference as a **static, version-agnostic placeholder** which is filled in
**client-side** at page load from `package-list.json` (already the source of truth, already same
origin, already what `history.js` reads):

```html
<span class="fhir-pb-dynamic" data-pb-version="4.0.0" data-pb-canonical="http://hl7.org.au/fhir">
  <span class="fhir-pb-superseded">The current version which supersedes this version is
     <a class="fhir-pb-current-link" href="…"><span class="fhir-pb-current-version">the current version</span></a></span>
  <span class="fhir-pb-iscurrent" style="display:none">This is the current published version</span>
  <span class="fhir-pb-prerelease" style="display:none">This version is a pre-release. …</span>
</span>
<script>/* fetch package-list.json, fill version+link, show the matching wording */</script>
```

A small inline script (self-contained, idempotent, no dependencies) reads `package-list.json`
same-origin (derived from the canonical path so it works under HTTPS without mixed-content), finds
the current milestone exactly as the server does (`current === true`, not the ci-build "current"
entry, `path` under canonical), fills in the version number + link, and shows the wording that
matches **this page's own version** (superseded / is-current / pre-release). The page's own version
is static and known at generation time, so the **emitted markup is byte-identical on every page of
every version regardless of which version is current**.

Result: a new milestone only updates `package-list.json` (one file). Old versions' pages are
unchanged, so the publisher no longer rewrites — and the site no longer re-uploads — the whole tree
on a milestone. The "current is vX" UX is preserved (filled by JS); the no-JS fallback shows the
"superseded … see the Directory of published versions" wording with a link to the canonical root.

The behaviour is **off by default** — only IGs that opt in via `publish-setup.json` get it, so this
is safe to land upstream without changing any existing site's output.

## Files

- `PublishBoxStatementGenerator.genFragment(...)` — new `dynamicCurrentVersion` overload + the
  `dynamicCurrentVersionBlock(...)` markup and the inline `CURRENT_VERSION_SCRIPT`.
- `PublicationProcess` (`-go-publish`) — reads `website.dynamic-publish-box`, threads it through
  `updatePublishBox(...)` to `genFragment(...)` at every call site (new version, past versions,
  current/root, technical-correction, withdrawal).
- `IGWebSiteMaintainer` / `IGReleaseUpdater` (`-publish-update`) — same flag plumbed through so site
  maintenance produces the same markup (won't revert it).

## Validation

Measured against a real `-web` mirror of three AU Base versions (4.0.0/4.1.0/5.0.0, 5,153 HTML
files) using `-publish-update`, which runs the identical per-version rewrite loop a milestone uses:

| Milestone run | publish-box behaviour | old-version HTML files rewritten |
|---|---|---|
| Baseline (flag off), current bumped | hard-coded current | **3,973 / 5,153** |
| 1st A2 milestone (flag on) — one-time migration | hard-coded → placeholder | 3,973 / 5,153 |
| **2nd A2 milestone (flag on), current bumped again** | placeholder (stable) | **0 / 5,153** |

After a one-time migration, subsequent milestones rewrite **zero** old-version pages.

## Notes / follow-ups

- The separate `addPageVersions` ("Page versions: R3 R4 R5 …") list is **not** addressed here; it
  still changes on old pages when a *new milestone folder* is added (it links the same page across
  milestone folders). That is a different mechanism and a smaller, less frequent effect; it could be
  made dynamic the same way in a follow-up if desired.
- Branch is cut from `fix/cloud-static-html-redirects` (PR #1327, the cloud/static-HTML redirect
  fix) — rebase onto master for submission.
