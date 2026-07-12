# Issue #11286 Reproduction

Issue: [Option to exclude drafts from PR counts](https://github.com/badges/shields/issues/11286)

Date reproduced: July 12, 2026

## Environment setup

- macOS on Apple Silicon
- Node.js v24.15.0 through nvm
- npm v11.12.1
- Fork: `azizu06/shields`
- Branch: `issue-11286-docs`
- Starting point: current `upstream/master` at `aa75efb72e`

I reused the fork and local setup from my first contribution cycle, then fetched the latest upstream changes and created a separate worktree for issue #11286. The existing checkout was still on my merged Cycle 1 branch and had unrelated untracked files, so using a new worktree kept the Phase II evidence isolated from both the old branch and the eventual code PR.

I followed the repository setup instructions with Node 24 and ran `npm ci` successfully. The install printed deprecation warnings and reported existing dependency vulnerabilities, but it completed without an installation error. I did not change dependencies because they are unrelated to this issue.

## Steps to reproduce

These requests were run back to back so the repository state was the same for both badge counts.

1. Request the current open pull-request badge for `badges/shields`.

   ```bash
   curl -fsSL \
     'https://img.shields.io/github/issues-pr/badges/shields.json'
   ```

2. Request the same badge with the proposed `excludeDrafts` query parameter.

   ```bash
   curl -fsSL \
     'https://img.shields.io/github/issues-pr/badges/shields.json?excludeDrafts'
   ```

3. Use GitHub search to confirm that the repository actually has both draft and non-draft open pull requests.

   ```bash
   curl -fsSL \
     -H 'Accept: application/vnd.github+json' \
     -H 'X-GitHub-Api-Version: 2022-11-28' \
     'https://api.github.com/search/issues?q=repo%3Abadges%2Fshields%20is%3Apr%20is%3Aopen%20draft%3Atrue'

   curl -fsSL \
     -H 'Accept: application/vnd.github+json' \
     -H 'X-GitHub-Api-Version: 2022-11-28' \
     'https://api.github.com/search/issues?q=repo%3Abadges%2Fshields%20is%3Apr%20is%3Aopen%20draft%3Afalse'
   ```

## Observed behavior

Both badge requests returned the same result:

```json
{ "label": "pull requests", "message": "26 open", "color": "yellow" }
```

At the same time, GitHub search reported 7 open draft PRs and 19 open non-draft PRs. Adding `?excludeDrafts` did not change the badge count, even though the repository had drafts that should have been excluded.

The exact live counts will change as pull requests are opened, closed, or marked ready for review. The repeatable part is that the unfiltered badge and the badge with `?excludeDrafts` return the same total while GitHub search shows at least one open draft.

## Expected behavior

The request with `?excludeDrafts` should count only open non-draft pull requests. For the captured repository state, the unfiltered request should report 26 and the filtered request should report 19.

## Code trace

The current route in `services/github/github-issues.service.js` declares only the `base` and path `pattern`. It has no `queryParamSchema` for `excludeDrafts` or `onlyDrafts`.

`GithubIssues.handle()` receives `variant`, `user`, `repo`, and `label`, then derives `isPR` and `isClosed`. It never reads a draft-related query parameter. The PR branch in `GithubIssues.fetch()` calls GraphQL's `repository.pullRequests(states:, labels:)` field and returns its `totalCount`. That query filters by state and label, but an open PR can still be a draft, so all open drafts remain in the count.

Shields' base service only validates and passes service-specific query parameters when the route has a `queryParamSchema`. Without one, the transformed service query parameters are empty. That is why the proposed parameter is ignored instead of changing the GitHub request.

## Files and functions involved

- `services/github/github-issues.service.js`
  - `GithubIssues.route`
  - `GithubIssues.fetch()`
  - `GithubIssues.handle()`
- `core/base-service/base.js`
  - query-parameter validation and transformation before `handle()`
- `services/github/github-issues-search.service.js`
  - existing analogous GraphQL `search` query that returns `issueCount`
- `services/github/github-issues.tester.js`
  - current issue and pull-request badge service tests
