# Counterbranch scanner Action

Compare static authorization coverage between two Git revisions and review the
result in your pull request. The scanner examines source without executing your
application. Findings are advisory; they do not prove runtime enforcement.

## What it checks

Counterbranch compares recognized authorization-related source evidence between
the pull request's merge base and head.

| Change or situation | What the report tells you | Why it matters |
| --- | --- | --- |
| A recognized guard or authentication check is added, removed, or changed | The affected operation's before-and-after static evidence | Review whether the change matches the intended access rules. |
| A role, permission declaration, or captured authorization hint changes | The declaration or evidence recorded by the scanner that changed, where recognized | Access-related code can change while the endpoint list stays the same. |
| An endpoint is added or removed | The added or removed operation, with revision-specific source links | Review access expectations alongside changes to the API. |
| An endpoint’s method, path, or recorded handler details change | Changed operation evidence, or separate added and removed operations when identities differ | A routing change can alter which code handles a request. |
| A recognized external policy artifact changes | The changed policy evidence or known policy path | Policy changes can need review even when application guards are unchanged. |
| Changed code is flagged as outside the operation model | The reported coverage gap and source location | Direct manual review to code the scanner cannot assess fully. |

Operation coverage includes recognized patterns in Express, NestJS, Spring MVC,
Django, Django REST framework, FastAPI, and Flask. Coverage varies by pattern;
listing a framework does not mean every application construct is modeled.

Unsupported or indirect forms can be unresolved or omitted. An empty result
does not prove that all application code was covered. Recognized guards and
permission declarations are not proof of effective runtime access.

## Results

The `outcome` output is one of:

| Outcome | Plain meaning |
| --- | --- |
| `CLEAN` | No review-triggering change was found within the supported comparison. |
| `NEEDS_OWNER_REVIEW` | Reported changes or relevant unmodeled source need review. |
| `INCOMPLETE` | The comparison records limits that prevent a complete assessment; missing deltas are not evidence of no change. |
| `UNKNOWN` | The Action could not establish a validated comparison result; inspect the failed stage in the run. |

Always inspect `has_incomplete` and the report. A successful Action step is not
merge approval, and `CLEAN` applies only to the supported static comparison.

## Status

Pre-release development. Use only versions listed in
[Releases](https://github.com/counterbranch/action/releases). Each
published release includes the tested commit, qualification scope, and a
workflow pinned to that commit.

## Requirements

- An `ubuntu-24.04` x86_64 GNU/Linux runner.
- A full checkout, using `actions/checkout` with `fetch-depth: 0` and
  `persist-credentials: false`.
- The absolute checkout path and exact lowercase, 40-character base and head
  commit IDs. The scanner compares their single merge base with the head.

## Inputs

| Input | Description | Default |
| --- | --- | --- |
| `repository` | Absolute path to the full checkout | Required |
| `base` | Target-side commit | Required |
| `head` | Candidate commit | Required |
| `profile` | Scan bounds: `default`, `large`, or `xl` | Default bounds |
| `timeout-seconds` | Whole comparison deadline, from 1 to 3600 seconds | `1800` |
| `post-comment` | Create or update the PR comment; accepts `true` or `false` | `false` |
| `report-retention-days` | Requested automatic report retention, from 1 to 400 days | `7` |

## Reports

`report_bundle` contains `comparison.json`, `report.md`, `REPORT-GUIDE.md`, and
`provenance.json`. When no validated comparison is available, the `UNKNOWN`
bundle omits `comparison.json` and retains the failure information.

With `post-comment: true`, the Action uploads the report ZIP and links it from
the PR comment. Download it and start with `REPORT-GUIDE.md` for further
exploration. GitHub artifact downloads require sign-in and repository read
access. Requested retention is subject to repository and organization limits;
artifacts can be deleted earlier. With commenting disabled, there is no
automatic upload; callers can retain `report_bundle` themselves.

## PR comments

Commenting supports same-repository `pull_request` events and requires
`pull-requests: write`. Fork PRs and other events skip comment publication. Do
not use `pull_request_target` to work around this boundary.

Use one publisher workflow per PR with this concurrency configuration:

```yaml
concurrency:
  group: counterbranch-scanner-${{ github.event.pull_request.number }}
  cancel-in-progress: false
```

Commenting therefore **requires** one publisher workflow per PR in the repository,
using the repository-wide, PR-keyed concurrency group above with
`cancel-in-progress: false`; other workflows should use `post-comment: false`.
Within each run, publisher jobs and Action calls for the same PR must run sequentially.
GitHub's comment API has no atomic ownership compare-and-swap: separately serialized
publisher workflows would still replace each other's reports. Ignoring these
constraints can cause a stale overwrite or duplicate comment that requires manual repair.
Publication errors remain Action failures.

## Versions

Action releases and scanner kit releases are versioned separately. This
snapshot uses the signed [scanner kit v0.41.3](https://github.com/counterbranch/alpha-releases/releases/tag/v0.41.3),
containing Counterbranch 0.41.3 and Discovery 0.24.0. The Action verifies the
kit's release identity, signature, checksum, size, and platform before running
its executables. No signup is required for this build.
