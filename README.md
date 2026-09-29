# Donmai native drift check

Analyze a pull request's complete diff without checking out or running its code. The Action downloads Donmai **v0.72.53**, verifies the archive against a SHA-256 embedded in [`action/run.py`](action/run.py), and runs `donmai arch assess --require-diff`. It produces a job summary, a check annotation, and, when permitted, a pull request comment.

Use the Action from a `pull_request` workflow and pin it to a reviewed full commit SHA:

```yaml
name: architecture drift
on: pull_request
permissions:
  contents: read
  pull-requests: read
jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: RenseiAI/donmai-drift-action@004ee04968c1543d6dfe1c36033b2bafb3326284
        with:
          comment: never
```

The read-only example disables comments. To let the Action publish or update its own comment, grant `pull-requests: write` and use the default `comment: auto`. No checkout step, model account, server, daemon, or additional credential is required by the Action. Its executable has an independent version and checksum pin; changing the Action ref does not select an arbitrary analyzer version.

## What it checks

This is native, regex-based diff analysis. It reports patterns, conventions, and decision signals; it does not learn an architectural baseline or run an LLM. The policy evaluates **observations**, not semantic architecture violations. A successful job means the selected policy passed on an available complete diff. It does not certify a design or prove that a change has no architectural problems.

| Input | Default | Meaning |
| --- | --- | --- |
| `token` | `github.token` | `pull-requests: read` to assess the diff; `pull-requests: write` to publish a comment. |
| `gate-policy` | `no-severity-high` | `none`, `no-severity-high`, `zero-deviations`, or `max:N`. |
| `comment` | `auto` | `auto`, `required`, or `never`. |

`no-severity-high` is the native CLI policy name: it gates when any observation has confidence at least **0.65**, not a semantic severity grade. `zero-deviations` gates on any observation; `max:N` gates above N observations; `none` reports without gating. Invalid policies fail rather than silently disabling the check. Missing or truncated diffs, unavailable GitHub access, changed pull request commits, download failures, and checksum failures all fail the job. They never become a clean assessment.

Outputs are `result` (`clean`, `gated`, `error`), `observations` (absent on error), `head-sha` (the event's head commit), and `comment-status` (`posted`, `updated`, `disabled`, `unavailable`). A successful assessment verifies both head and base before and after fetching the diff. The comment includes the head SHA so a later pull request update cannot make an old assessment look current.

## Forks and permissions

Use the `pull_request` event. The Action refuses `pull_request_target` and other events. It never checks out the pull request head, runs repository scripts, loads project configuration, or evaluates text from a diff. It executes in an empty temporary directory with an isolated home and only the required GitHub authentication. Python runs in isolated mode so inherited `PYTHONPATH` and user-site modules cannot execute during Action startup.

GitHub normally gives fork pull requests and Dependabot read-only tokens. With `comment: auto`, a denied comment write produces a warning and `comment-status: unavailable`; the assessment and job summary still run. Do not give fork code a write token to obtain a comment. Use `comment: required` when comment publication is an explicit requirement: unavailable permission then fails the job. Organization policy may also restrict comments on same-repository pull requests.

Only a comment carrying this Action's marker **and** authored by `github-actions[bot]` is updated. Otherwise a new comment is created. A custom PAT is not needed; custom-token comments may accumulate because the Action does not edit comments by human accounts. API redirects refuse rather than forward a credential to a new destination. The Action currently supports github.com, Linux/macOS x64 and ARM64, Python 3 with a working CA certificate store, and `gh`. GitHub-hosted Ubuntu runners provide these tools. Windows and GitHub Enterprise hosts are not supported yet.

Comments and annotations contain only validated commit identity, policy, and locally counted observation totals. Pull request titles, paths, bodies, diff content, and raw analyzer/network errors are deliberately excluded to avoid mentions, markup, workflow-command injection, and credential disclosure.

## Maintainer verification

Run the local tests with an already verified v0.72.53 executable:

```sh
DONMAI_ACTION_TEST_BINARY=/path/to/donmai python3 -I -m unittest discover -s action -v
```

The suite drives the real executable against a local fake `gh` transport: complete diff, gated diff, unavailable diff, missing patch, and hostile pull request text. Separate controls exercise the embedded checksum, archive links/traversal, event and commit identity, comment ownership/permission errors, annotation escaping, and the actual metadata launcher. The tests do not contact GitHub or post comments. The `test` workflow obtains its analyzer by calling this Action's own checksum-verifying installer before running all 14 tests.

The initial Action files were sourced byte-exact from Donmai v0.72.48. This focused repository now maintains its Action files independently; its analyzer version and archive digests are pinned separately in `action/run.py`. The focused repository does not assert Marketplace availability or outside-organization adoption. For reports about vulnerabilities in the Donmai analyzer, follow [Donmai's public security policy](https://github.com/RenseiAI/donmai/blob/main/SECURITY.md).

GitHub references: [composite Action metadata](https://docs.github.com/en/actions/reference/workflows-and-actions/metadata-syntax), [least-privilege `GITHUB_TOKEN` permissions](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token), and [pull request workflow security](https://docs.github.com/en/actions/reference/security/securely-using-pull_request_target).
