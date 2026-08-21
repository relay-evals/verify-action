# Relay verify

Block a pull request whose changed lines have no verified evidence.

```yaml
permissions:
  contents: read
  id-token: write          # Relay exchanges this for a 15-minute signing credential

steps:
  - uses: actions/checkout@v4
    with: { fetch-depth: 0 }   # required — see "Why fetch-depth: 0" below

  - uses: relay-evals/verify-action@v1
```

That is the whole setup. **There is no secret to add.** The job proves its own
identity with an OIDC token, Relay verifies it against GitHub's published keys,
and issues a signing credential that expires in fifteen minutes and is bound to
that one run. The private key is generated inside the job and never leaves it.

## What it checks

**Every changed line executed under signed evidence.** Not "the suite passed" —
that a receipt, signed and content-bound to this exact worktree, shows your
changed lines actually running. A verdict recomputes from those receipts
offline; anyone can re-derive it without trusting the run that produced it.

**That your tests depend on your change.** A test can execute a changed line
without checking anything:

```js
test("verifyPayment", () => { verifyPayment(input); });   // covered. proves nothing.
```

That satisfies changed-line coverage completely, and it is the first shape an
agent optimising for a green check will find. Relay rebuilds your repository as
it was before the branch, keeps your new tests, and runs them. Tests that
genuinely cover a change *fail* there. Tests that pass there proved nothing.

Reported as a `VACUOUS_TESTS` finding. It does **not** fail the job unless you
ask it to — see `fail-on-vacuous-tests`.

## Why `fetch-depth: 0`

"Before the branch" is the merge base with your target branch. `actions/checkout`
clones one commit by default, which has no common ancestor to compare against.
Without full history the check reports `inconclusive` and tells you why.

It does not fall back to `HEAD`. On a branch whose change is already committed,
`HEAD` *contains* the change — that counterfactual would be identical to the
change, every honest test would pass against it, and every honest test would be
called vacuous. A check that guesses confidently is worse than one that declines.

## Adopting it without breaking anyone

Both enforcement switches default so that installing this action cannot turn a
passing build red on day one.

```yaml
- uses: relay-evals/verify-action@v1
  with:
    fail-on-verdict: false        # observe the gate before enforcing it
```

Watch it for a week. Then set `fail-on-verdict: true`, add the check to branch
protection, and — separately, once you have seen what it finds —
`fail-on-vacuous-tests: true`.

## Inputs

| input | default | |
|---|---|---|
| `version` | pinned | CLI version. Pin it; a build that pins nothing installs something new the morning after a release. |
| `server` | `https://relayevals.com` | |
| `working-directory` | `.` | |
| `fail-on-verdict` | `true` | Whether BLOCK or UNRESOLVED fails the job. |
| `red-first` | `true` | Whether to check that changed tests depend on the change. |
| `fail-on-vacuous-tests` | `false` | Whether tests that pass without the change fail the job. |

## Outputs

| output | |
|---|---|
| `verdict` | `PASS`, `BLOCK`, `UNRESOLVED` or `POLICY_DRIFT` |
| `changed-total` | Changed surfaces considered |
| `changed-verified` | Of those, how many some receipt covers |
| `red-first` | `genuine`, `vacuous`, `inconclusive` or `skipped` |

## What this does not tell you

**Relay never says a change is safe to merge.** A PASS means every changed line
executed under signed evidence and the changed tests depend on the change. It
says nothing about whether your assertions are *right* — a test asserting the
wrong expected value still fails without the change, and is correctly scored
genuine. Mutation testing is the check for that, and it is not shipped.

`UNRESOLVED` means Relay could not decide, not that your change is bad. Treat it
as a configuration problem.

## How it works

A **composite** action, not a JavaScript one, and that is a security decision as
much as a convenience. A JS action ships a bundled `dist/index.js` — tens of
thousands of lines of vendored dependencies committed to a repository, which
every consumer runs with their OIDC token in scope and almost nobody reads.

This runs the published CLI and nothing else, so what executes in your job is
the same artifact you can download, checksum, and build from source:

```
curl -fsSL https://relayevals.com/releases/<version>/install.sh | bash
```

Every release publishes its own installer carrying its version, tarball URL and
tarball sha256 as literals, so pinning the installer pins the whole chain.

Docs: https://relayevals.com/docs
