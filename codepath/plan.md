# Implementation Plan — Issue #10162

**Issue:** [badges/shields#10162](https://github.com/badges/shields/issues/10162) **Approach:** Migrate the Azure DevOps **build** badge from SVG-scraping to the authenticated Azure DevOps REST API, matching the pattern already used by the coverage / tests / release badges. (The maintainer confirmed this direction in the issue thread.)

## UMPIRE

### Understand

The build badge doesn't send the user's PAT, so it breaks on private Azure DevOps projects. It must authenticate like the other Azure DevOps badges.

### Match

`AzureDevOpsCoverage` (`services/azure-devops/azure-devops-coverage.service.js`) is the working template: it `extends AzureDevOpsBase`, calls `getLatestCompletedBuildId(...)`, then fetches an authenticated JSON endpoint and renders. The build badge should follow the same shape.

### Plan

1. Change `AzureDevOpsBuild` to **`extends AzureDevOpsBase`** so it inherits `static auth` and the authenticated `fetch`.
2. Replace the SVG scrape with a JSON call to `GET /_apis/build/builds?definitions={definitionId}&$top=1&branchName=refs/heads/{branch}` via the base's authenticated `fetch`.
3. Read `value[0].status` + `value[0].result` and map them to the existing build-status badge output (reuse `renderBuildStatusBadge`).
4. **`stage` / `job` — DECIDED: keep them (option a).** Currently supported via `?stage=` / `?job=`; the builds-list API returns whole-build status only, so per-stage/job parity needs Azure's **Timeline API** (`/_apis/build/builds/{buildId}/timeline`, which returns records typed `Stage` / `Job` with their own `result`).
   - **Rationale:** the maintainer's stated concern is replicating existing functionality via the API (see Risks), so preserving stage/job parity outranks the "only copy siblings" heuristic. Dropping it would regress behavior the issue did not ask to remove.
   - **Shape:** reuse `getLatestCompletedBuildId(...)` for the build id (same as siblings); when `stage`/`job` is given, fetch the Timeline and read the matching record's `result`; otherwise use the overall build `result`.
   - Fallback **(b)** (scope out, raise with maintainer) only if review pushes back on PR size.

5. Add `azure-devops-build.spec.js` — the build badge currently has only a `.tester.js`, no unit spec.

### Implement

Phase III, on branch `fix-issue-10162`.

### Review

Self-review against shields `CONTRIBUTING.md`:

- PRs are squash-merged — no need to pre-squash my own commits.
- **Service changes must include tests.**
- **PR title must tag the affected service in square brackets** so CI runs those tests, e.g. `[AzureDevops] Use PAT auth for the build badge on private projects`.
- Commit author/message cannot be amended after merge.

### Evaluate

shields tests services two ways; I'll use both:

- **Unit tests (`*.spec.js`, `nock`):**
  - assert that `Authorization: Basic …` **is** sent when a token is configured (the inverse of the reproduction);
  - assert correct mapping of Azure `result` values (`succeeded` / `failed` / `canceled` / `partiallySucceeded`) to badge output.
- **Service tests (`*.tester.js`):** keep ≥1 live "picture check" green.
- Run `npm test` to confirm no regressions.
- **No real Azure account needed** — `nock` supplies the fake responses.

## Risks / open questions

- **stage/job parity** — the maintainer's explicit concern about replicating existing functionality via the API. **Resolved (step 4): keep stage/job via the Timeline API.**
- **`result` → badge mapping** — reuse shields' existing build-status mapping for consistency with the other CI badges.
