# Configuration

The workflows read all configuration from GitHub: secrets (`secrets.*`) and variables (`vars.*`). No value is stored in the repository. Names only are listed here.

## AWS access (all deploy and rollback workflows)

| Name | Kind | Used by | Purpose |
| --- | --- | --- | --- |
| `AWS_ROLE_ARN` | variable | stage, prod, rollback | IAM role to assume through GitHub OIDC |
| `DEV_AWS_ROLE_ARN` | variable | dev | IAM role for the dev ECS jobs |
| `DEV_FRONTEND2_ROLE_AMPLIFY_BUILD` | variable | dev | IAM role for the Amplify release |
| `AWS_REGION` | variable | all deploy workflows | Region for the AWS calls |

There are no AWS access keys. The workflows request `id-token: write` and exchange the GitHub OIDC token for short-lived credentials.

## Registry

`GITHUB_TOKEN` (provided by GitHub) logs in to GHCR to push and read images.

## Frontend build

| Name | Kind | Passed as |
| --- | --- | --- |
| `NODE_AUTH_TOKEN` | secret (variable as fallback) | BuildKit secret `node_auth_token` in the stage and prod image builds. The dev build runs on the runner and passes it as an environment variable to `npm ci` |
| `STRIPE_SECRET_KEY` | secret | BuildKit secret `stripe_secret_key` in the stage and prod image builds. An environment variable on the dev runner build |
| `REACT_APP_*` / `NEXT_PUBLIC_*` (API URL, WebSocket URL, vendor API URL, Maps key, Auth0 domain, client ID, scope and audience, Stripe publishable key, PostHog key and host) | secret or variable, depending on environment | Build arguments. These are public values that end up in the browser bundle |
| `CSP_*` (API domain, script, connect, frame, font, media, style, analytics sources, manifest domain, enabled flag) | variable | Build arguments |
| `NEXT_PUBLIC_PARTNER_INTEGRATION_ENABLED` | variable | Build argument |
| `SENTRY_*` (DSN, environment, organisation, project, traces sample rate, auth token) | variable | Build arguments |

In the stage and prod image builds, the two BuildKit secrets are mounted only for the `npm install` and `npm run build` steps (`frontend-2/Dockerfile`), so they are not written to an image layer.

## Notifications and tracking

| Name | Kind | Used by | Purpose |
| --- | --- | --- | --- |
| `SLACK_DEPLOY_ENABLED` | variable | prod, rollback | When `true`, a chat notification is sent |
| `SLACK_DEPLOY_WEBHOOK` | secret | prod, rollback | Webhook URL for that notification |
| `TRACKER_API_TOKEN` | secret | dev | Token for the issue tracker status update |

## Environments

Jobs reference the GitHub environments `dev`, `stg` and `prod`. Which of the values above are stored at repository level and which per environment, and any required reviewers, are set in the repository settings and are not part of these files.

## Where runtime configuration comes from

These files cover building and deploying only. How the running ECS tasks receive their own configuration and secrets is defined outside them.
