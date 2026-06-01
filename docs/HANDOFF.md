# Handoff & automation setup

This tap's only moving part is `.github/workflows/bump-formula.yml`, which opens a PR when
upstream cuts a newer stable Verus release. It authenticates as a **GitHub App**, not a
personal token, for two reasons:

1. A PR opened by the default `GITHUB_TOKEN` does **not** trigger other workflows (GitHub's
   anti-recursion rule), so `tests.yml` would never run on the bot PR, and `main`'s branch
   protection requires the `ci-passed` check, so the PR could never merge. An App token is
   not the default token, so its PR **does** trigger CI.
2. An App is owned by the **repo/org**, not a person, so the automation keeps working after a
   maintainer change. A personal PAT dies with the account that made it.

The workflow reads two secrets and never names who owns the App, so a handoff is just
"recreate the App and repopulate the secrets", with no workflow edits.

| Secret        | What it holds                                      |
| ------------- | -------------------------------------------------- |
| `TAP_APP_ID`  | the GitHub App's numeric **App ID**                |
| `TAP_APP_KEY` | the App's **PEM private key** (full file contents) |

## First-time setup (current personal owner)

1. **Create the App**: GitHub → Settings → Developer settings → GitHub Apps → New GitHub App.
   - Name: anything (e.g. `verus-tap-bot`). The workflow derives the bot identity from the
     App slug at runtime, so the name is not baked into any file.
   - Homepage URL: the tap repo URL (any valid URL is accepted).
   - Uncheck **Webhook → Active** (this App takes no events).
   - **Repository permissions:** `Contents: Read and write`, `Pull requests: Read and write`.
     Nothing else.
   - Where can it be installed: "Only on this account" is fine.
2. **Generate a private key**: on the App's page, "Private keys" → Generate. Download the
   `.pem`.
3. **Install the App**: App page → Install App → install on `yipjunkai/homebrew-verus` only.
4. **Add the secrets**: repo → Settings → Secrets and variables → Actions:
   - `TAP_APP_ID` = the App ID shown at the top of the App's settings page.
   - `TAP_APP_KEY` = the entire contents of the downloaded `.pem` (including the
     `-----BEGIN/END-----` lines).
5. **(Optional) Enable auto-merge**: repo → Settings → General → "Allow auto-merge". The
   workflow queues each bump PR with `gh pr merge --auto`; with this on, a PR self-merges once
   `ci-passed` is green. With it off, the PR is simply left for manual merge (the workflow does
   not fail).
6. **Test**: Actions → "bump formula" → Run workflow. With the tap already current it should
   no-op cleanly (`changed=false`); when behind, it opens a PR that CI runs on.

## Transferring to verus-lang (recommended path: org makes its own App)

This avoids depending on undocumented App-transfer credential behavior: the org starts from a
known-good credential and the original maintainer drops out entirely.

1. Transfer the repository to `verus-lang` (Settings → General → Danger Zone → Transfer).
   Repo-level secrets and branch protection move with the repo, but the App installation does
   not follow, so:
2. An org owner repeats **First-time setup** steps 1-5 above, owned by and installed on
   `verus-lang/homebrew-verus`, writing the App ID and private key into the **same secret
   names** (`TAP_APP_ID`, `TAP_APP_KEY`).
3. Revoke/delete the old personal App: the org's App fully replaces it. No workflow change is
   needed; the YAML only ever references the secret names.

Alternative (lower-setup, less clean): transfer the App itself via its settings → Advanced →
Transfer ownership to `verus-lang`. The App keeps its ID/installations/key, so the secrets stay
valid, but the org inherits an App configured by the previous maintainer rather than one they
created and reviewed. Prefer the recreate path above unless that tradeoff is acceptable.
