# Reproduction — Issue #10162

**Issue:** [badges/shields#10162 — Azure DevOps – Build Badge – PAT Token not used on private projects](https://github.com/badges/shields/issues/10162) **Branch:** `fix-issue-10162` **Author:** Abduaziz Umarov ([@azizu06](https://github.com/azizu06))

## Summary

shields.io serves several Azure DevOps badges. Three of them — coverage, tests, and release — authenticate with a user-provided Personal Access Token (PAT) and therefore work on **private** Azure DevOps projects. The **build** badge does not: it never sends the PAT. So on a project that blocks anonymous access, the build badge fails while its siblings on the same project succeed.

## Environment

|  |  |
| --- | --- |
| OS | macOS (Apple Silicon), Darwin 25.5 |
| Node.js | v24.15.0 (via nvm) |
| npm | 11.x |
| Repo | fork `azizu06/shields`, branch `fix-issue-10162`, off `badges/shields@master` |

**Setup notes / gotchas (for the next contributor):**

- shields' default branch is **`master`**, not `main`.
- `.npmrc` sets `engine-strict=true`, and a dev dependency (`lint-staged`) requires Node **`>=22.22.1`**. Installing on Node 22.19 aborts with `EBADENGINE` and leaves `node_modules` empty. Node 24 works.
- Native modules (e.g. `re2`) are compiled for a specific Node version. The badge server must **run on the same Node version used to install** (Node 24), or it crashes at startup with `ERR_DLOPEN_FAILED` / `NODE_MODULE_VERSION` mismatch.

## Why this is a code/behavior reproduction (not a live private project)

The literal symptom requires a private Azure DevOps org with a completed pipeline and a PAT. Microsoft gates free CI parallelism behind a manual grant that can take several business days, so reproducing the exact end-user scenario is impractical for this milestone. Instead I reproduce the **mechanism**: I confirm the build badge never attaches the PAT, while its siblings do. For a missing-auth bug, demonstrating the absent auth path is a faithful reproduction.

## Steps to reproduce

### A. The badge works for a PUBLIC project — anonymously (no token)

1. Install dependencies on Node 24: `nvm use 24 && npm ci`.
2. Start the badge server: `npm run start:server` (listens on port `8080`).
3. Open in a browser: `http://localhost:8080/azure-devops/build/totodem/8cf3ec0e-d0c2-4fcd-8206-ad204f254a96/2.svg`
4. **Result:** the badge renders **`build | passing`**. It works with **no token configured**, because the example project (`totodem`) is public.

### B. Trace the code to show no token is ever sent

5. `services/azure-devops/azure-devops-build.service.js:38` — `class AzureDevOpsBuild extends BaseSvgScrapingService`. This base scrapes an SVG image and declares **no `static auth`**.
6. The same file's `handle()` (≈ lines 110–132) fetches `https://dev.azure.com/{org}/{projectId}/_apis/build/status/{definitionId}` through the helper imported on line 9.
7. `services/azure-devops/azure-devops-helpers.js:16–18` — that helper calls `serviceInstance._requestSvg(...)`. There is **no `authHelper`** anywhere in this path, so no `Authorization` header is ever attached.
8. Contrast `services/azure-devops/azure-devops-base.js:15–24` — `class AzureDevOpsBase extends BaseJsonService` declares `static auth = { passKey: 'azure_devops_token', authorizedOrigins: ['https://dev.azure.com'] }` and fetches via `this._requestJson(this.authHelper.withBasicAuth(...))`. `withBasicAuth` attaches the PAT as an HTTP Basic `Authorization` header.
9. `services/azure-devops/azure-devops-coverage.service.js:44` — `class AzureDevOpsCoverage extends AzureDevOpsBase`: a sibling badge that goes through the authenticated path.

## Observed vs. expected

- **Observed:** the build badge fetches its data through `BaseSvgScrapingService._requestSvg`, a path with no authentication. Even when `azure_devops_token` is configured, no `Authorization` header is sent, so any project that blocks anonymous access returns an error and the badge fails.
- **Expected:** like coverage / tests / release, the build badge should send the configured PAT (HTTP Basic auth) so it can read build status for private projects.

## Root cause

The bug is **structural**, not a missing line inside one function. In shields, authentication is determined by the base class a service extends and the `_request*` method it calls:

- The **build** badge extends the **SVG-scraping** base and reads Azure's anonymous status _image_ (`/_apis/build/status/...`), so it has no `static auth` and never calls `authHelper`.
- Every **other** Azure DevOps badge extends **`AzureDevOpsBase`** (JSON + `authHelper.withBasicAuth(...)`), so the PAT is attached and they work on private projects.

The scraped image endpoint is anonymous-only and can't be authenticated, so the fix is to move the build badge onto the authenticated JSON API. The base class already implements an authenticated `/_apis/build/builds` call in `getLatestCompletedBuildId()` (`azure-devops-base.js:33`).

**Primary file to change:** `services/azure-devops/azure-devops-build.service.js`.
