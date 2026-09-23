# auto-trigger-wp-updates-renovate

> **This is an example repository.** It is a plain [Roots Bedrock](https://roots.io/bedrock/) WordPress project used to experiment with automatic dependency pull requests via [Renovate](https://docs.renovatebot.com/) on **GitHub** and **Bitbucket Cloud**. It is not a production site.
>
> The Dependabot version of the same experiment lives in [tobiasnygren/auto-trigger-wp-updates](https://github.com/tobiasnygren/auto-trigger-wp-updates).

## What it demonstrates

Renovate checks the Composer dependencies daily (00:00–06:59 UTC) and opens pull requests for:

- **WordPress core** (`roots/wordpress`): minor and patch releases only. Major versions (e.g. 7.x) are disabled.
- **Plugins and themes** (`wp-plugin/*`, `wp-theme/*`): grouped into a single PR, majors included (`separateMajorMinor: false`).
- **Other Composer packages**: one PR per package.

The policy lives in [`renovate.json`](renovate.json):

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "schedule": ["* 0-6 * * *"],
  "constraints": {
    "php": "8.3"
  },
  "packageRules": [
    {
      "description": "constraints.php mirrors config.platform.php in composer.json; don't bump it",
      "matchDepTypes": ["tool-constraint"],
      "enabled": false
    },
    {
      "matchPackageNames": ["roots/wordpress"],
      "matchUpdateTypes": ["major"],
      "enabled": false
    },
    {
      "matchPackageNames": ["wp-plugin/*", "wp-theme/*"],
      "groupName": "wordpress plugins and themes",
      "separateMajorMinor": false
    }
  ]
}
```

Validate changes with:

```sh
npx --yes --package renovate renovate-config-validator --no-global renovate.json
```

Because `renovate.json` is already on the default branch, Renovate treats the repo as onboarded. It skips the onboarding PR and goes straight to update PRs.

## GitHub: Mend Renovate app

1. Install the [Mend Renovate app](https://github.com/apps/renovate) and give it access to this repository only.
2. Wait for the next run inside the schedule window, or tick **Create all awaiting schedule PRs at once** in the Dependency Dashboard issue to get them now. Job logs are at [developer.mend.io](https://developer.mend.io/).

## Bitbucket Cloud: self-hosted in Pipelines

Mend also has a hosted app for Bitbucket Cloud. This repo self-hosts instead, with [`bitbucket-pipelines.yml`](bitbucket-pipelines.yml), to test that path.

1. **Credentials.** Create an [Atlassian API token](https://support.atlassian.com/bitbucket-cloud/docs/create-an-api-token/) for the Renovate user with these scopes:
   - `read:repository:bitbucket`
   - `write:repository:bitbucket`
   - `read:pullrequest:bitbucket`
   - `write:pullrequest:bitbucket`
   - `read:user:bitbucket`
   - `read:workspace:bitbucket`

   App passwords stopped working in June 2026, so don't use one.
2. **Workspace group.** Add the Renovate user to a workspace group that has the **Create repositories** permission. Renovate uses this to check reviewer membership.
3. **Repository variables** (Repository settings → Pipelines → Repository variables, all **Secured**):

   | Variable | Value |
   |---|---|
   | `RENOVATE_BB_EMAIL` | Atlassian account email of the Renovate user |
   | `RENOVATE_BB_API_TOKEN` | The API token |
   | `RENOVATE_GITHUB_COM_TOKEN` | *(optional)* read-only github.com token, for release notes |

4. **Run it.** Go to Pipelines → Run pipeline → `custom: renovate`. For a daily run, add a schedule under Pipelines → Schedules that fires inside 00–06 UTC.

To test outside the schedule window, set `IGNORE_SCHEDULE` to `true` when you start the run.

## Keeping GitHub and Bitbucket in sync

GitHub is the source of truth. [`.github/workflows/mirror-to-bitbucket.yml`](.github/workflows/mirror-to-bitbucket.yml) pushes `main` to Bitbucket on every push to GitHub `main`. You can also run it by hand from the Actions tab.

- It needs the Actions secret `BITBUCKET_API_TOKEN`: an Atlassian API token with the `write:repository:bitbucket` scope.
- Only `main` is mirrored. Each Renovate instance manages its own `renovate/*` branches.
- Merge Renovate PRs on GitHub only. After the mirror runs, Bitbucket's Renovate sees the update is already on `main` and closes its matching PR.
- The push isn't forced. If you commit or merge directly on Bitbucket `main`, the mirror fails instead of overwriting that work.

## Gotchas

| Problem | Fix |
|---|---|
| Every update fails with "requirements could not be resolved" | The locked packages need PHP 8.3. Keep `require.php: ">=8.3"` and `config.platform.php: "8.3"` in `composer.json`, plus `constraints.php` in `renovate.json`. |
| Should I add credentials for `repo.wp-packages.org`? | No. It's a public Composer repo and needs no `hostRules`. |
| `composer.lock` hash out of date after editing `composer.json` | Run `composer update --lock` and commit the lock file. |
| Bitbucket 401 errors | Use an Atlassian **API token** with your account **email** as the username. App passwords are gone. |
| No Dependency Dashboard on Bitbucket / `WARN: Cannot ensure issue` | Bitbucket Cloud removed Issues in August 2026. `RENOVATE_DEPENDENCY_DASHBOARD=false` isn't enough because `config:recommended` turns it back on, so the pipeline sets it through `RENOVATE_FORCE`. |
| No PRs, and the log says `Skipping branch creation as not within schedule` | Working as intended outside 00:00–06:59 UTC. Use the dashboard checkbox on GitHub, or `IGNORE_SCHEDULE=true` on Bitbucket. |
| "Update php tool constraint to v8.x" PR | Renovate treats `constraints.php` as a dependency. It's disabled with a `matchDepTypes: ["tool-constraint"]` rule so it stays in sync with `config.platform.php`. |
