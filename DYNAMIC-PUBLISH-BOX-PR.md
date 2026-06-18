# Opt-in dynamic (client-side) publish box — zero-churn milestones

Makes the per-version "publish box" banner resolve its cross-version references **client-side** so a
new milestone no longer rewrites the publish box in every page of every past version. Opt-in, off by
default. (Companion PR: **dynamic source viewers** / `dynamic-source-viewers` — orthogonal, separate
branch.)

## Problem

Every published IG page carries a publish-box banner. Two parts of it name *other* versions and are
**baked into the HTML of every page of every past version**:

1. the current-version reference — *"The current version which supersedes this version is **6.0.0**"*;
2. the **"Page versions: R3 R4 R5 …"** cross-link list (`addPageVersions`).

Cutting a new milestone changes (1) for every page, and publishing a new milestone *folder* changes
(2) — so the publisher **rewrites the publish box in every page of every prior version**. For a large
multi-version IG that is tens of thousands of files rewritten and re-uploaded on every milestone — the
single biggest cost of a milestone release (diff to sync, build time, and a long history of fragile
milestone runs).

## Change

Add an **opt-in** mode (`publish-setup.json` → `website.dynamic-publish-box: true`). When set, both
references are emitted as **static, version-agnostic placeholders** filled in **client-side** at page
load from `package-list.json` (already the source of truth, same origin, already what `history.js`
reads). One small inline script (self-contained, idempotent, one `fetch`) drives both:

```html
<span class="fhir-pb-dynamic" data-pb-version="4.0.0" data-pb-canonical="http://hl7.org.au/fhir">
  <span class="fhir-pb-superseded">The current version which supersedes this version is
     <a class="fhir-pb-current-link" href="…"><span class="fhir-pb-current-version">the current version</span></a></span>
  <span class="fhir-pb-iscurrent" style="display:none">This is the current published version</span>
  <span class="fhir-pb-prerelease" style="display:none">This version is a pre-release. …</span>
</span>
… For a full list of available versions, see the Directory of published versions
<span class="fhir-pb-page-versions" data-pb-version="4.0.0"></span>
<script>/* fetch package-list.json: fill current version + link, toggle wording; build Page-versions */</script>
```

- **Current-version**: the script finds the current milestone exactly as the server does
  (`current === true`, not the ci-build "current" entry, `path` under canonical), fills the version
  number + link, and shows the wording matching **this page's own version** (superseded / is-current /
  pre-release). The page's own version is static, so the markup is **byte-identical on every page of
  every version regardless of which version is current**.
- **Page versions**: the script lists `package-list.json` milestones and `HEAD`-probes each for this
  page, showing only milestones that actually contain it (matching the server) — so adding a new
  milestone folder never rewrites old pages either.

Result: a new milestone only updates `package-list.json` (one file); old versions' pages are
unchanged, so the publisher no longer rewrites — and the site no longer re-uploads — the tree on a
milestone. The "current is vX" UX and the Page-versions nav are preserved (filled by JS); the no-JS
fallback shows the "superseded … see the Directory of published versions" wording with a link to the
canonical root.

Off by default — only IGs that opt in get it, so it cannot change any existing site's output.

## Files

- `PublishBoxStatementGenerator.genFragment(...)` — new `dynamicCurrentVersion` overload + the
  `dynamicCurrentVersionBlock(...)` markup and the inline `CURRENT_VERSION_SCRIPT` (handles both the
  current-version reference and the page-versions list in one fetch).
- `IGReleaseVersionUpdater` — `dynamicPublishBox` ctor flag; `addPageVersions` emits the placeholder.
- `PublicationProcess` (`-go-publish`) — reads `website.dynamic-publish-box`, threads it through
  `updatePublishBox(...)`/`IGReleaseVersionUpdater` to `genFragment(...)` at every call site (new
  version, past versions, current/root, technical-correction, withdrawal).
- `IGWebSiteMaintainer` / `IGReleaseUpdater` (`-publish-update`) — same flag plumbed through so site
  maintenance produces the same markup (won't revert it).

## Validation

Measured against a real `-web` mirror of three AU Base versions (4.0.0/4.1.0/5.0.0, 5,153 HTML files)
using `-publish-update`, which runs the identical per-version rewrite loop a milestone uses. Changed
HTML counted two independent ways (md5 manifest + the publisher's own `countUpdated`); they agree.

| milestone run | old-version HTML files rewritten |
|---|---|
| Baseline (flag off), current bumped | **3,973 / 5,153** |
| 1st milestone (flag on) — one-time migration (hard-coded → placeholder) | 3,973 / 5,153 |
| **2nd milestone (flag on), current bumped again** | **0 / 5,153** |
| flag on, **new milestone folder published** (page-versions) | **0 / 5,153** (was 2,554 before the page-versions fix) |

After a one-time migration, subsequent milestones rewrite **zero** old-version pages — including when a
new milestone folder is added.

## Headline (AU Base, ~27 versions)

| | today | next build (1st w/ flag) | every build after |
|---|---|---|---|
| **Milestone** — old-version pages rewritten | ~35,000 | ~35,000 (one-time migration) | **0** |
| **Milestone** — tree touched / uploaded | whole tree ~4.6 GB / ~88k objects | whole tree (migration) | **just the new version + root** |
| **Milestone** — build time | full-tree sync + rewrite-all + render | ~same (migration) | **≈ a working build** |
| **Working** — anything | (working never touched old versions) | unchanged | unchanged |

The first milestone after enabling the flag is a one-time migration (rewrites all old pages once,
hard-coded → placeholder); the second and every milestone after rewrite **zero**. (`old-version churn
35k → 0` is measured on 3 versions / 5,153 files and extrapolated ×27 ≈ 45,700 HTML.) Not addressed
here: the go-publish double-render (~15 min) — a separate lever (a `-reuse-build` flag) that bounds
per-version render time.

## Notes / follow-ups

- `website.dynamic-publish-box` needs **no schema change**: `publish-setup.json` has no schema — the
  publisher reads `website.*` keys ad-hoc and ignores unknown keys — so this PR is self-sufficient.
  (Optional: mention the key in the publish-setup.json example/docs.)
- Companion PR **dynamic source viewers** (`dynamic-source-viewers`) is orthogonal (render pipeline,
  per-version size) and lives on its own branch stacked on this one.
- Branch is cut from `fix/cloud-static-html-redirects` (PR #1327) — rebase onto master for submission.
