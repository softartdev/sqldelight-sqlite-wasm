# `gh` CLI Cheat Sheet

## Check Workflow Status

To check the status of the last run for a specific workflow, use the following command:

```bash
gh run list --workflow=<workflow_file_name> --limit=1
```

**Example for `web.yml`:**

```bash
gh run list --workflow=site.yaml --limit=1
```

## Cancel a Workflow Run

To cancel a running workflow, you first need to get the `RUN_ID` of the job.

1.  **List running jobs to find the `RUN_ID`:**

    ```bash
    gh run list --workflow=<workflow_file_name>
    ```

## View details for a run

Get the full details for a run (replace <RUN_ID>):

```bash
gh run view <RUN_ID> --web   # opens the run in your browser
# or
gh run view <RUN_ID>         # prints details in the terminal
```

2.  **Cancel the run using the `RUN_ID`:**

```bash
gh run cancel <RUN_ID>
```

## Trigger `site.yaml` manually (workflow_dispatch)

This repository's `site.yaml` supports manual dispatch. Trigger a run from the current branch:

```bash
gh workflow run site.yaml
```

To dispatch on a specific ref (branch or tag):

```bash
gh workflow run site.yaml --ref main
```

If you need to pass inputs (none are defined for this workflow), use `--field name=value`.

## Download logs for a run

```bash
gh run download <RUN_ID> --logs
# extracts logs to ./<RUN_ID>/logs/
```

## Get deployed Pages URL

The `site.yaml` workflow sets the environment URL when it deploys GitHub Pages. To find the URL after a successful deploy, open the deploy run in the browser and look for the environment outputs, or use the run view:

```bash
gh run view <RUN_ID> --web
```

If you deployed using the `actions/deploy-pages` step, the page URL is usually shown in the workflow summary.

## Notes & tips

- `gh` must be authenticated to access private workflows or to perform cancel/dispatch actions. Run `gh auth status` to check.
- The `site.yaml` workflow builds the JS browser distribution via Gradle (`./gradlew :jsBrowserDistribution`) and uploads the artifact located at `build/dist/js/productionExecutable` for Pages deployment.
- Common troubleshooting: if the workflow fails during Gradle build on CI, run the same Gradle task locally to reproduce: `./gradlew :jsBrowserDistribution --stacktrace`
