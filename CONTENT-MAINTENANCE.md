# Content Maintenance Guide — eds-da-demo

Tested on 17 Sep 2026 against the live setup: GitHub `venkateshKA-volkswagen/eds-da-demo` + Document Authoring (da.live) + `*.aem.page` / `*.aem.live`. Every flow below was executed and verified, not assumed.

## Environments

| Environment | URL | Content shown |
|---|---|---|
| Author | https://da.live/#/venkateshka-volkswagen/eds-da-demo | Latest saved source |
| Preview | https://main--eds-da-demo--venkateshka-volkswagen.aem.page/ | Last **Preview**ed version |
| Live | https://main--eds-da-demo--venkateshka-volkswagen.aem.live/ | Last **Publish**ed version |
| Local dev | http://localhost:3002 (`aem up` in this repo) | Local code + previewed content |

## 1. Updating an existing page (tested)

1. Open the page in the editor: da.live → browse to the site → click the page (e.g. `index`). Direct URL pattern: `https://da.live/edit#/venkateshka-volkswagen/eds-da-demo/<page>`.
2. Edit in place (WYSIWYG). **Saving is automatic** — the change lands in the source ~5–10 seconds after you stop typing (cloud icon top-right shows sync state). There is no Save button.
3. Nothing is visible on preview or live yet — see Preview/Publish below.

Rollback (tested): the editor keeps version history, and the source API accepts a full-page overwrite. Restoring a previous body and re-previewing returned the page to its exact prior state.

## 2. Simple text changes (tested)

Click into any text and type. Verified end to end: a sentence appended to a homepage paragraph appeared in the DA source after autosave, then on `.aem.page` and `localhost:3002` immediately after Preview, while `.aem.live` kept the old text until Publish.

## 3. Links and internal references (tested)

- **Internal links**: use site-relative paths (`/`, `/about`, …). Verified: they pass through the pipeline unchanged, so they resolve correctly on localhost, preview, and live alike. Never hardcode `.aem.page`/`.aem.live` hostnames in content links.
- **External links**: pass through untouched (verified with `https://www.aem.live/docs/`).
- In the editor, select text and use the link tool (chain icon) in the left toolbar.
- Renaming or moving a page does **not** update pages that link to it — search the site for the old path before renaming.

## 4. Image and media references (tested)

- Authors paste or drag images into the editor; DA stores them and the page references them.
- Verified pipeline behavior: an `<img>`/`<picture>` reference (even an absolute URL to another EDS site) is ingested into the media bus on Preview and rewritten to a local, hashed, auto-optimized reference — `./media_<sha>.jpg?width=…&format=webply&optimize=medium` — with responsive webp/jpeg variants generated automatically. No manual image optimization needed for authored content.
- The same image content always hashes to the same `media_…` URL, so re-using an image adds no weight.
- Images committed to the **git repo** (icons etc.) are NOT auto-optimized — only authored media is.

## 5. Preview behavior (tested)

- Edits are **gated**: after autosave, `.aem.page` still served the old version until Preview was clicked (verified).
- Trigger: open the page in the editor → paper-plane button (top right) → **Preview**. DA also opens the preview URL for you.
- Propagation was effectively immediate (page CDN cache is `max-age=60`, so allow up to a minute for a cached path).
- The local dev server (`aem up`) serves **previewed** content — it updated at the same moment as `.aem.page`.
- API equivalent (needs an authorized login, see §7): `POST https://admin.hlx.page/preview/venkateshka-volkswagen/eds-da-demo/main/<path>`. Unauthenticated preview is rejected (verified: HTTP 401), because building the preview reads from the DA content source.

## 6. Publishing behavior (tested)

- Publish is a **separate, explicit step**: editor → paper-plane → **Publish** (or `POST https://admin.hlx.page/live/.../<path>`). It promotes the *previewed* copy to `.aem.live`.
- Verified sequence on a test page: create → Preview (`.aem.page` 200, `.aem.live` 404-equivalent) → Publish → `.aem.live` 200 with identical content. Homepage edits stayed preview-only until published.
- The initial site seeding (da.live start flow) published everything once; that is why `.aem.live` serves content even though no one clicked Publish afterwards.

### ⚠️ Findings to act on

1. **Anonymous publish is currently allowed.** `POST /live/...` on the admin API succeeded with **no authentication** (verified HTTP 200): anyone who knows the org/site name can promote already-previewed content to live. Acceptable for a sandbox/demo; before real use, configure site authentication / admin roles (see https://www.aem.live/docs/authentication-setup-site and related admin-permission docs).
2. **Nobody currently holds delete/unpublish rights.** Unpublish (`DELETE /live/...`) and unpreview (`DELETE /preview/...`) returned **403 for anonymous *and* for the site owner's login**, including through DA's own delete dialog (it calls the same API). Consequence, verified: **deleting a page in DA does not remove it from preview or live** — the copies keep serving as orphans. Until delete permissions are configured, treat publishing as effectively irreversible and never publish drafts or test pages. (`/maintenance-test` on this site is such a leftover: source, preview, and live copies exist; remove them once delete rights are set up.)
3. To "unpublish" content today, the workaround is to overwrite the page with replacement content and re-preview + re-publish it.

## 7. Roles, permissions and access needed for content maintenance (identified)

| Task | Requirement |
|---|---|
| Edit / create / delete content in DA | Adobe ID login (this org federates to Škoda/VW SSO). The DA org `venkateshka-volkswagen` grants write on `CONFIG` and `/ + **` to `DZCIIVQ@skoda-auto.com` (see da.live → org settings). Add teammates there before they can author. |
| Trigger Preview | Same DA-authorized login (anonymous is 401 — the preview build reads the DA source). |
| Trigger Publish | **Currently no login required** (finding #1 above). |
| Unpublish / unpreview | **Currently nobody** (finding #2 above) — requires setting up admin/delete permissions for the site. |
| Read preview/live sites | Public — no authentication configured. |
| Change code (blocks, styles, scripts) | GitHub write access to `venkateshKA-volkswagen/eds-da-demo`; the AEM Code Sync app must keep this repo in its selected-repositories list. |
| Change content source / site config | Repo write access (`fstab.yaml`) — this site is fstab-based and intentionally **not** registered in the `api.aem.live` config service. Do not complete the `tools.aem.live` "AEM Configuration Setup" wizard for this repo: it would move the site to the config service, where our tokens currently have no role. |
| Local development | Clone of this repo + `npx @adobe/aem-cli up` (no credentials needed; content comes from the public preview). |

## Quick reference — author loop

1. Edit at da.live (autosaves).
2. **Preview** → check `https://main--eds-da-demo--venkateshka-volkswagen.aem.page/<page>` (and localhost).
3. **Publish** → goes to `https://main--eds-da-demo--venkateshka-volkswagen.aem.live/<page>`.
4. Inspect what the pipeline delivers with `curl <url>/<page>.plain.html`.
