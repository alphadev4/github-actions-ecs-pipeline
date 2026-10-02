# Workflows

All paths are under `.github/workflows/`. Resource names such as `sample-app-prod-ecs-cluster` are placeholders for the real ones.

## Pull request checks

| Workflow | Jobs | Notes |
| --- | --- | --- |
| `CI-Backend.yml` | `gitleaks`, `lockfile`, `lint`, `audit` | gitleaks scans only the commits in the PR (`origin/<base>..HEAD`). `lockfile` runs `uv lock --locked` and fails if `pyproject.toml` and `uv.lock` have drifted. `audit` runs a pinned `pip-audit` on the exported lock file, with an explicit and commented ignore list. |
| `backend-pytests.yml` | `test` | `pytest -n auto` against a `postgres:16` service container. Runs on PRs and pushes to `dev`, `staging` and `main`, and on demand. |
| `CI-Frontend.yml` | `gitleaks`, `lint`, `test` | eslint, prettier, type-check and vitest, in the `frontend/` directory. |
| `pr-security-scan.yml` | `security` | Calls a reusable workflow from the organisation's shared workflow repository. That workflow is not part of this extract. |

CI workflows use a concurrency group per ref with `cancel-in-progress: true`, so a new push cancels the superseded run.

## Deploy workflows

| Workflow | Trigger | Jobs, in order |
| --- | --- | --- |
| `new-dev-cicd.yml` | Push to `release/*`, or manual | `changes` -> `build-and-push-backend` and `deploy-frontend-amplify` -> `backend-ecs-update`, `celery-worker-ecs-update`, `celery-beat-ecs-update` -> `update-tracker-status` |
| `new-stg-cicd.yml` | Manual | `changes` -> `build-and-push-backend` and `build-and-push-frontend` -> four `*-ecs-update` jobs |
| `prod-cicd.yml` | Manual, with a `version` input | `resolve-version` -> `changes` -> `build-and-push-backend` and `build-and-push-frontend` -> four `*-ecs-update` jobs -> `create-release` |
| `rollback-prod.yml` | Manual, with `version` and `services` inputs | `verify-images` -> `rollback-backend`, `rollback-celery-worker`, `rollback-celery-beat`, `rollback-frontend` -> `notify` |

### Change detection

`dorny/paths-filter` decides what to build and deploy:

- `backend/**` rebuilds and redeploys the API, Celery worker and Celery beat services.
- `frontend-2/**` rebuilds and redeploys the frontend.
- On a manual dev run, everything is treated as changed.
- In prod the comparison base is the previous release tag. If there is no previous tag, everything is deployed.

### Image tags

| Environment | Backend image | Frontend image |
| --- | --- | --- |
| Dev | `ghcr.io/<repo>-backend-new-dev:<sha>` | Built and released through Amplify |
| Stage | `ghcr.io/<repo>-backend-new-stg:<sha>` | `ghcr.io/<repo>-frontend-new-stg:<sha>` |
| Prod | `ghcr.io/<repo>-backend-prod:<version>` and `:<sha>` | `ghcr.io/<repo>-frontend-prod:<version>` and `:<sha>` |

The ECS deploy step is given the tag: the commit SHA in dev and stage, the release version in prod.

### ECS deployment

Each ECS job assumes an IAM role through GitHub OIDC (`aws-actions/configure-aws-credentials`) and then runs `donaldpiret/ecs-deploy` against one ECS service with `rollback: true` and a timeout (300 seconds in stage and prod, 900 in dev). The four services per environment are the API, the Celery worker, the Celery beat scheduler and the frontend (the dev frontend uses Amplify instead).

### Concurrency

`prod-cicd.yml` and `rollback-prod.yml` share the group `prod-deploy` with `cancel-in-progress: false`. A deploy and a rollback queue behind one another and never run at the same time.

### Release creation

`create-release` runs only if no deploy job failed or was cancelled and at least one succeeded. It creates the git tag and the GitHub release from the deployed commit, and posts a chat notification when the `SLACK_DEPLOY_ENABLED` variable is `true`.
