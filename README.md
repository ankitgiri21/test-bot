# AutomationHQ TestBot — GitHub Actions Runner

Run an AutomationHQ Test Bot from a GitHub Actions workflow, on demand, and get
the pass/fail results as a rich report right on the workflow's Summary page —
no AutomationHQ dashboard access needed to see how a run went.

This repo is a **template**. You don't need to write any code — you only need
to:
1. Drop in your own Test Bot configuration.
2. Add your AutomationHQ JWT token as a GitHub secret.
3. Run the workflow.

---

## 1. Prerequisites

- A GitHub account (this repo can live under your personal account or an org).
- An AutomationHQ account with at least one **Test Bot** already created.
- Access to generate a **JWT token** for your AutomationHQ account (see step 4).

---

## 2. Use this template

1. Click **Use this template** (top of the repo page) → **Create a new repository**,
   or fork the repo — either works.
2. Give it any name and make it **private** if your Test Bot configuration or
   results are sensitive.
3. Clone it, or just edit files directly on github.com — both are fine, since
   the only two files you need to touch don't require any local tooling.

---

## 3. Get your Test Bot configuration JSON

Every AutomationHQ Test Bot has a configuration (browser, OS, grid, custom
properties, etc.) that the executor API needs in order to run it.

> **Note:** Get this from your AutomationHQ dashboard — open the Test Bot you
> want to run and use its **Export / Copy Configuration** option to get the
> exact JSON. If you don't see that option, ask your AutomationHQ contact for
> the configuration JSON for your Test Bot; the shape must match the template
> in this repo (`testBotId`, `executionConfiguration`, etc.).

Keep this JSON handy for the next step.

---

## 4. Replace `configs/testbot-config.json`

Open [`configs/testbot-config.json`](configs/testbot-config.json) in this repo.
It currently looks like this (placeholders in `<ANGLE_BRACKETS>`):

```json
{
  "name": "<YOUR_TEST_BOT_NAME>",
  "testBotId": "<YOUR_TEST_BOT_ID>",
  ...
}
```

Replace the **entire file's contents** with the real JSON you got in step 3,
then commit the change. The workflow reads this file on every run, so it
always executes whatever Test Bot configuration is currently committed here.

---

## 5. Generate a JWT token

> **Note:** Generate this from your AutomationHQ account settings / API
> access section. Treat it like a password — anyone with it can trigger Test
> Bot executions on your account.

Copy the token value — you'll paste it once into GitHub in the next step and
won't need to touch it again unless it expires or you regenerate it.

---

## 6. Add the token as a GitHub secret

1. In your new repo, go to **Settings** → **Secrets and variables** → **Actions**.
2. Click **New repository secret**.
3. Name: `TESTBOT_JWT_TOKEN`
4. Value: paste the JWT token from step 5.
5. Click **Add secret**.

The workflow reads this secret automatically — it is never printed in logs or
committed to the repo.

---

## 7. Run the workflow

1. Go to the **Actions** tab of your repo.
2. Select the **Run TestBot** workflow in the left sidebar.
3. Click **Run workflow** → **Run workflow** (this is a manual/`workflow_dispatch`
   trigger — it won't run automatically on push).
4. Click into the running job to watch it live, or wait for it to finish.

A typical run:
- Loads and validates `configs/testbot-config.json`.
- Triggers the Test Bot execution against AutomationHQ.
- Polls every 5 seconds until the execution reaches a terminal state
  (succeeded, completed, failed, or cancelled).
- Downloads the detailed results and turns them into a report.

---

## 8. View the results

- **Summary page**: open the finished workflow run — the **Summary** tab
  shows execution ID, status, and a full per-suite/per-script/per-step
  breakdown with ✅/❌ markers, generated directly on the page. This is the
  fastest way to see what happened without leaving GitHub.
- **Checks tab**: a `TestBot Results` check is published with pass/fail counts
  per test case.
- **Artifacts**: the raw JSON, a JUnit XML (`results/junit-results.xml`), and
  the markdown report are all uploaded as a downloadable artifact named
  `testbot-results-<run number>` at the bottom of the run page.

If the run fails before any results exist (bad config, bad token, unreachable
API, timeout), the Summary page still shows a **Reason** row explaining
exactly what went wrong — check that first before digging into logs.

---

## Troubleshooting

| Symptom in the Summary "Reason" row | What it means | Fix |
|---|---|---|
| `configs/testbot-config.json not found` | File missing or renamed | Make sure the file exists at that exact path |
| `... is not valid JSON` | Pasted JSON has a syntax error | Re-copy the config JSON from AutomationHQ, don't hand-edit the structure |
| `... is missing a "testBotId" field` | Config JSON is incomplete | Make sure the full exported JSON was pasted, not a partial snippet |
| `TESTBOT_JWT_TOKEN secret is not set` | Secret wasn't added, or has a different name | Re-check step 6 — the secret name must be exactly `TESTBOT_JWT_TOKEN` |
| `Trigger request failed (curl exit ...)` | API rejected the request — commonly an expired/invalid token, or a `testBotId` that doesn't belong to your account | Regenerate the JWT token (step 5) and re-check the `testBotId` in your config |
| `Polling timed out after 20 minutes` | The execution is taking longer than expected, or is stuck | Check the Test Bot's status directly in the AutomationHQ dashboard |
| `Execution finished ... but fetching detailed results failed` | Execution completed but the results endpoint errored | Re-run; if it persists, contact AutomationHQ support with the Execution ID shown in the summary |

---

## FAQ

**Can I run this automatically instead of manually?**
Yes — add a `schedule:` or `push:` trigger under the `on:` key in
[`.github/workflows/testbot5.yml`](.github/workflows/testbot5.yml). It ships
with only `workflow_dispatch` (manual) so nothing runs unexpectedly before
you've configured your secret.

**Can I run multiple Test Bots from one repo?**
This template runs the single Test Bot described in
`configs/testbot-config.json`. For multiple bots, duplicate the workflow file
and config file per bot (e.g. `testbot-config-checkout.json` +
a matching workflow), or open an issue if you'd like a matrix-based version.

**Is my JWT token safe here?**
Yes — it's stored as an encrypted GitHub Actions secret, injected only as an
environment variable inside the run, and is never echoed to logs or written
to any committed file.
