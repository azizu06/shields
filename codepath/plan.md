# Issue #11286 Solution Plan

Issue: [Option to exclude drafts from PR counts](https://github.com/badges/shields/issues/11286)

This plan follows UMPIRE and the direction the Shields maintainers gave in the review of [PR #11401](https://github.com/badges/shields/pull/11401).

## Understand

The `issues-pr` badge counts every open pull request, including drafts. A user who wants to show how many PRs are ready for review cannot exclude drafts, so the badge can overstate the actionable review queue.

The existing `GithubIssues` service owns four issue variants and four pull-request variants. It uses separate `repository.issues` and `repository.pullRequests` GraphQL fields internally, but exposes them through one service class. Its route has no draft query parameters, and the `repository.pullRequests` connection does not provide a draft-state filter.

The completed change should meet these acceptance criteria:

1. Existing issue badge URLs and behavior stay unchanged.
2. Existing PR badge URLs stay unchanged.
3. PR badges accept presence-only `excludeDrafts` and `onlyDrafts` parameters.
4. `excludeDrafts` adds `draft:false` to the GitHub search.
5. `onlyDrafts` adds `draft:true` to the GitHub search.
6. Unfiltered PR badges still count all matching PRs.
7. Open/closed, raw/non-raw, and optional label variants keep their existing badge labels and messages.
8. Repository-not-found behavior remains clear.
9. Existing tests pass and new tests prove the draft filtering behavior.

## Match

### Maintainer-requested direction

The original PR added draft parameters to the combined `GithubIssues` service and moved both issues and PRs onto search. The maintainer pointed out that draft issues are not a real concept and asked for a cleaner split:

- `GithubIssues` should keep `issues`, `issues-raw`, `issues-closed`, and `issues-closed-raw` on the existing `repository` query.
- A new `GithubPullRequests` service should own `issues-pr`, `issues-pr-raw`, `issues-pr-closed`, and `issues-pr-closed-raw`, plus `excludeDrafts` and `onlyDrafts`.
- All PR variants should use the GraphQL `search` query so filtered and unfiltered PR counts have one data path.

### Analogous code

`services/github/github-issues-search.service.js` is the closest existing match. Its `BaseGithubIssuesSearch.fetch()` method calls GraphQL `search(query:, type: ISSUE)` and reads `issueCount`. `GithubRepoIssuesSearch.handle()` also shows how Shields prefixes a user-supplied search with `repo:{user}/{repo}`.

The closed PR #11401 is useful prior work rather than a patch to merge unchanged. It already explored the search query, repo-not-found detection, parameter tests, and badge-label changes. The new implementation should fetch that work for reference, address the pending review comments, and preserve attribution to the original author in the eventual code commit as the maintainer requested.

### Git history

`git log --follow -- services/github/github-issues.service.js` shows that the service has existed for years and was migrated to GraphQL in commit `a5b2f5436c`. `git blame` shows the current combined route variants were last grouped together in `095e4f889a`. This is established behavior, so the split should preserve the public URLs instead of renaming them.

## Plan

### 1. Separate issue and PR ownership

Update `services/github/github-issues.service.js` so it keeps only the four issue variants. Remove its PR-only response schema, `isPRVariant` map, pull-request fetch branch, and PR rendering decision. Keep the existing repository issue query and issue badge output unchanged.

Create `services/github/github-pull-requests.service.js` for the four existing `issues-pr` variants. The new class will keep the same `base: 'github'` and the same public PR route patterns, so current badge URLs continue to work even though a different service class handles them.

### 2. Validate draft parameters

Add a route `queryParamSchema` for `excludeDrafts` and `onlyDrafts`. Follow Shields' presence-only flag convention with `Joi.equal('')`, matching the maintainer's review on PR #11401. Treat the two flags as mutually exclusive with Joi's `oxor` validation because asking for only drafts and excluding drafts at the same time is contradictory.

Document both parameters in the new service's OpenAPI definitions. Draft terminology should appear only on the PR badge documentation.

### 3. Build one GitHub search path for PRs

Construct the search string from controlled parts:

```text
repo:{user}/{repo} is:pr is:{open|closed} [label:"..."] [draft:false|draft:true]
```

Use GraphQL `search(query: $query, type: ISSUE) { issueCount }`, following `github-issues-search.service.js`. Use this search path for every PR variant, even when neither draft flag is present, as requested by the maintainer.

Keep a lightweight `repository(owner:, name:)` field in the GraphQL request if needed to distinguish a missing repository from a valid search with zero results. Reuse `transformErrors` and the existing GitHub authentication base class so error handling and credentials remain consistent with neighboring services.

### 4. Preserve rendering behavior

Move the PR-specific rendering behavior into the new service without changing existing output:

- non-raw variants use the `pull requests` label and add `open` or `closed` to the message;
- raw variants use `open pull requests` or `closed pull requests` as the label and return only the metric;
- label-filtered variants keep the label name in the badge label.

When one draft flag is present, make the badge label clearly say whether it counts drafts or non-drafts. Keep multiword label quoting consistent with the current service.

### 5. Split and expand tests

Keep issue-only cases in `services/github/github-issues.tester.js`. Create `services/github/github-pull-requests.tester.js` and move the current PR cases there so the service ownership is visible in the test layout.

Add deterministic tests that inspect the outbound GraphQL search and return controlled counts for:

1. open PRs without a draft filter;
2. open PRs with `excludeDrafts`, proving `draft:false` is sent;
3. open PRs with `onlyDrafts`, proving `draft:true` is sent;
4. raw and non-raw output;
5. closed PRs;
6. a single-word and multiword label;
7. zero matching PRs;
8. repository not found;
9. both draft flags supplied, expecting query-parameter validation to reject the request.

Retain at least one live picture check if that matches the current Shields service-test convention, but do not depend on changing live PR counts for the exact draft-filter assertions.

### 6. Keep the eventual PR clean

The `codepath/` reproduction and plan files belong on this Phase II documentation branch only. Start Phase III from a clean branch based on current upstream and bring over the original contributor's relevant code with attribution. Do not include course planning files in the upstream PR, based on the feedback from my first contribution cycle.

## Implement

Implementation begins in Phase III on a separate `fix-issue-11286` branch. Build the split in small commits so the service move, search behavior, and tests are reviewable independently. Do not copy PR #11401 blindly because it predates the maintainer's requested separation and current upstream changes.

## Review

Before opening a PR:

- compare every changed route against the current public issue and PR variants;
- inspect the diff for unrelated formatting or course files;
- confirm parameter names and presence-only syntax match Shields conventions;
- confirm the previous contributor receives the attribution requested by the maintainer;
- self-review against `CONTRIBUTING.md` and the repository PR template;
- explain why the service split and search query are necessary before describing the code changes.

## Evaluate

Run focused formatting, lint, unit, and GitHub service tests first. Then run the repository's required broader checks before submission.

The final verification should prove all three counts against controlled data:

- unfiltered PR count includes drafts;
- `excludeDrafts` returns only non-drafts;
- `onlyDrafts` returns only drafts.

Also confirm the four issue variants still use the repository issue query and produce the same outputs as before. Record the commands and results in the contribution README so Phase III and Phase IV evidence remain traceable.

## Tradeoffs and open questions

- GitHub search is necessary because `repository.pullRequests` cannot filter by draft status. It also gives the PR service one consistent query path, but search has different behavior and limits than the repository connection, so tests must cover error and zero-result cases.
- Splitting the classes adds one service file and one tester file, but it removes issue-only terminology from PR logic and prevents meaningless draft options from appearing on issue badges.
- Rejecting both draft flags is clearer than silently choosing one or building a contradictory search. If the maintainer prefers another behavior, keep it as an explicit review decision and add a test for the chosen rule.
- Search qualifiers are assembled from route values, so label quoting and escaping need focused coverage, especially for spaces, quotes, and slash characters.
