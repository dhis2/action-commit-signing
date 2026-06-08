# action-commit-signing

A DHIS2 composite GitHub Action that configures git to **SSH-sign commits**, so
commits created later in the same job (including by tools like
[`semantic-release`](https://github.com/dhis2/action-semantic-release)) are
signed and show as **Verified** on GitHub.

Use it in any repository whose protected branch enforces a *"Commits must have
verified signatures"* rule.

## How it works

The action runs as a step in your job and only sets up git configuration. It
does not create commits itself. Any commit made by a later step in the same job
inherits this configuration and is signed automatically:

1. Writes the private signing key to `$RUNNER_TEMP` (`chmod 600`).
2. Derives the public key from it (no separate public-key secret needed).
3. For a passphrase-protected key, loads it into an `ssh-agent` so signing never
   prompts; for a passphraseless key, no agent is needed.
4. Sets `gpg.format=ssh`, `user.signingkey`, `commit.gpgsign=true`, plus the
   commit author name/email and an `allowed_signers` file for local
   verification.

Because the action configures the git CLI rather than wrapping a specific tool,
it works with manual `git commit`, `semantic-release`, and anything else that
shells out to git.

> Note: a reusable workflow (`workflow_call`) cannot do this, because it runs as
> a separate job on its own runner; the git config would not reach your release
> job. That is why this is a composite action invoked with `uses:`.

## Prerequisites

For commits to show as **Verified** (and to pass a signed-commits rule), the
**public half** of the signing key must be registered on the GitHub account
named in `git-user-email` as a **Signing Key** (type `signing`, not
`authentication`), under *Settings -> SSH and GPG keys*. The commit email must
match an email on that account.

For DHIS2 this is the [`dhis2-bot`](https://github.com/dhis2-bot) account, with
the private key stored as the organization secret `DHIS2_BOT_SSH_SIGNING_KEY`.

## Usage

Add the action as a step **before** the step that creates commits, and make sure
the job has already checked out the repository.

```yaml
- uses: actions/checkout@v4
  with:
    token: ${{ secrets.DHIS2_BOT_GITHUB_TOKEN }}

- uses: dhis2/action-commit-signing@v1
  with:
    ssh-signing-key: ${{ secrets.DHIS2_BOT_SSH_SIGNING_KEY }}

# ... later steps that commit (e.g. semantic-release) are now signed
```

### Example: signing `semantic-release` commits

```yaml
- uses: dhis2/action-commit-signing@v1
  with:
    ssh-signing-key: ${{ secrets.DHIS2_BOT_SSH_SIGNING_KEY }}

- uses: dhis2/action-semantic-release@master
  with:
    publish-github: true
    github-token: ${{ secrets.DHIS2_BOT_GITHUB_TOKEN }}
```

### Passphrase-protected key

```yaml
- uses: dhis2/action-commit-signing@v1
  with:
    ssh-signing-key: ${{ secrets.DHIS2_BOT_SSH_SIGNING_KEY }}
    ssh-signing-key-passphrase: ${{ secrets.DHIS2_BOT_SSH_SIGNING_PASSPHRASE }}
```

## Inputs

| Input                        | Required | Default          | Description                                                                                       |
| ---------------------------- | -------- | ---------------- | ------------------------------------------------------------------------------------------------- |
| `ssh-signing-key`            | yes      |                  | Private SSH signing key.                                                                          |
| `ssh-signing-key-passphrase` | no       | `''`             | Passphrase for the key. Leave empty for a passphraseless key.                                     |
| `git-user-name`              | no       | `dhis2-bot`      | Commit author/committer name.                                                                     |
| `git-user-email`             | no       | `apps@dhis2.org` | Commit author/committer email. Must match the account holding the signing key for Verified status. |
| `global`                     | no       | `true`           | Apply git config globally (`true`) or to the current repository only (`false`).                   |

## Security notes

- The private key only ever exists in `$RUNNER_TEMP`, which is wiped when the
  ephemeral runner is torn down. The key is never printed and is masked as a
  secret in logs.
- Only the private key is required; the public key is derived at runtime.
- Pin the action to a tag (`@v1`) or commit SHA in consuming workflows.

## License

[MIT](./LICENSE)
