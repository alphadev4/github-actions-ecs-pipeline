# Deploy, roll back and troubleshoot

## Deploy

**Dev.** Push to a `release/*` branch, or run **Dev CI/CD** from the Actions tab. The backend image is built and the services are updated. The frontend is built on the runner and released through Amplify.

**Stage.** Run **Stage CI/CD** from the Actions tab on the branch to deploy. Only the parts that changed (backend or frontend) are built and deployed.

**Prod.**
1. Run **Prod CI/CD** from the Actions tab on the commit you want to release.
2. Enter a version in the form `vX.Y` (for example `v7.0`).
3. If the tag does not exist, the dispatched commit is deployed and tagged once every deploy succeeds. If the tag exists, that tag is re-deployed.
4. Approve the run if the `prod` environment requires it. Approval rules are set in the repository settings and are not part of these files.

## Roll back

1. Run **Rollback Prod** from the Actions tab.
2. Enter the version to roll back **to** (for example `v6.1`). Its images must already be in the registry.
3. Choose `all`, `backend-only` or `frontend-only`.

The workflow first checks that the image for that version exists, then redeploys it to the selected services.

## Troubleshoot

| Symptom | Where it comes from | What to do |
| --- | --- | --- |
| `Version must look like v7.0` | `resolve-version` in `prod-cicd.yml` rejects any version not matching `vX.Y` | Re-run with a version such as `v7.1` |
| `MISSING — cannot roll back to <version>` | `verify-images` in `rollback-prod.yml` could not find the image for that version | Check the version exists as a release; `all` and `backend-only` check the backend image, `all` and `frontend-only` check the frontend image |
| A prod run sits waiting | The `prod-deploy` concurrency group queues runs | Wait for the active deploy or rollback, or cancel it |
| Gitleaks fails on a PR | A secret pattern was found in the PR commits | Output is redacted. Rotate the value, then remove it from the PR commits |
| `lockfile` fails | `pyproject.toml` changed without a matching `uv.lock` | Run `uv lock` and commit the result |
| `audit` fails | `pip-audit` found a vulnerability not in the ignore list | Upgrade the package, or add a justified ignore |
| An ECS update job fails or times out | `ecs-deploy` could not stabilise the service within the timeout | Open that job's log. The step is configured with `rollback: true`. Check the ECS service events in AWS |
| Dev Amplify job fails | The Amplify job finished as `FAILED` or `CANCELLED` | The log shows the job status. Open the Amplify console for the failed build step |
| No release or tag created after a prod deploy | `create-release` is skipped when any deploy job failed or was cancelled | Fix the failing job and re-run with the same version |
| A service was not redeployed | The change filter found no changes in its directory | On dev, a manual run treats everything as changed. Otherwise change a file under that directory |

The workflows write to the Actions log only; they do not create job summaries.
