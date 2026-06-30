# CI/CD Deployments for TunnelHub Automations

Source: https://docs.tunnelhub.io/cli/ci-cd-deployments
Last captured: 2026-06-30

Use this reference when configuring automatic deployment for an integration built with `@tunnelhub/sdk`. The expected output is a GitHub Actions workflow at `.github/workflows/deploy-tunnelhub.yml`. Do not run `tunnelhub deploy-automation` from the local agent environment; include that command only inside the workflow file.

## Goal

Generate or update `.github/workflows/deploy-tunnelhub.yml` so GitHub Actions can publish a TunnelHub automation with `@tunnelhub/cli`.

The CI/CD flow must use account-level CI/CD credentials with an environment allowlist. These credentials belong to the account, but can only deploy to the TunnelHub environment UUIDs authorized when the credential was created.

## Prerequisites

Before generating the workflow, confirm or document that the project has:

- An automation already created in TunnelHub.
- A `tunnelhub.yml` configured for the automation being published.
- A `package.artifact` path in `tunnelhub.yml`; the workflow must generate the ZIP at this path before deploying.
- The UUID for each TunnelHub environment that the workflow can deploy to.
- Access to `Administration > Settings > CI/CD` in TunnelHub to create CI/CD credentials.
- GitHub repository permissions to create environments, secrets, and variables.

## Create the CI/CD Credential

In the TunnelHub portal:

1. Open `Administration > Settings`.
2. Open the `CI/CD` tab.
3. Click `Criar credencial CI/CD`.
4. Enter a credential name.
5. Select the authorized TunnelHub environments.
6. Finish creation.
7. Copy the `Client ID` and `Client secret`.

Important rules:

- Do not use `tunnelhub login` in CI.
- Do not commit credentials.
- Store `Client ID` and `Client secret` as CI provider secrets.
- Prefer least-privilege credentials, ideally one credential per environment for production-sensitive flows.
- The `Client secret` is displayed only once.
- If a secret is lost or exposed, regenerate or revoke it in `Administration > Settings > CI/CD`.

## GitHub Secrets and Variables

Recommended GitHub Actions configuration:

- `TH_CLIENT_ID`: GitHub Environment Secret.
- `TH_CLIENT_SECRET`: GitHub Environment Secret.
- `TH_ENVIRONMENT_ID`: GitHub Environment Variable containing the TunnelHub environment UUID.
- `TH_API_HOST`: Repository variable, GitHub Environment Variable, or fixed workflow value. Use `https://api.tunnelhub.io` for production.

The GitHub Environment and TunnelHub Environment are different concepts:

- GitHub Environment controls approvals, protection rules, secrets, and variables in GitHub.
- TunnelHub Environment is the UUID passed to `--environment-id` during deploy.
- The link between them is the `TH_ENVIRONMENT_ID` variable.

Example mapping:

- GitHub Environment `production` has `TH_ENVIRONMENT_ID=<production-environment-uuid>`.
- GitHub Environment `development` has `TH_ENVIRONMENT_ID=<development-environment-uuid>`.

## Workflow Generation Rules

When asked to configure automatic deploy, generate or update `.github/workflows/deploy-tunnelhub.yml`.

Required behavior:

- Use `workflow_dispatch` so deploy can be run manually.
- Use branch triggers only when the user asks for them or the branch mapping is clear from the project.
- Install dependencies using the existing package manager, detecting `pnpm-lock.yaml`, `yarn.lock`, or falling back to `npm ci`.
- Generate the ZIP artifact at the path configured by `package.artifact` in `tunnelhub.yml` before running deploy.
- Install `@tunnelhub/cli` in the workflow.
- Run `tunnelhub deploy-automation` only inside the workflow.
- Pass `--environment-id "$TH_ENVIRONMENT_ID"`.
- Pass a meaningful `--message`; this is required.

Recommended deploy message context:

- Commit message.
- Short commit SHA.
- Commit author email or GitHub actor.
- Branch name.
- Workflow run or PR/release context when useful.

The created revision stores:

- `createdBy` as `api-client:<clientId>`.
- `message` with the human and operational context supplied to `--message`.

## Conditional Test Safety

Tests are not mandatory for TunnelHub deploy workflow generation. Do not add a test step unless the project already has that convention, the workflow already runs tests, or the user explicitly asks for tests.

If the workflow will run tests, inspect `package.json` and the test structure before keeping or adding the test step. Avoid running debug, unmocked, or real-data tests in CI.

Check for:

- Test paths or filenames containing `unmocked`.
- Tests created for manual debugging.
- Tests that use real credentials or environment IDs.
- Tests that make real HTTP/database/service calls without mocks.

For Jest projects that keep real-call tests under an `unmocked` path, prefer scripts like:

```json
"scripts": {
  "test": "jest --testPathIgnorePatterns=unmocked",
  "test:coverage": "jest --collect-coverage --silent --testPathIgnorePatterns=unmocked",
  "test:badges": "jest --coverage --silent --testPathIgnorePatterns=unmocked && make-coverage-badge"
}
```

Do not rewrite unrelated test setup just because an `unmocked` path exists. Adjust or warn only when those tests would be invoked by the deploy workflow, or when the user asks for the test scripts to be made safe.

If it is unclear whether a test makes real calls, ask for confirmation or exclude the suspicious tests from the CI command instead of silently assuming they are safe.

## GitHub Actions Template: main to production

Use this when `main` should publish to the GitHub Environment `production`.

```yaml
name: Deploy TunnelHub Automation

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  deploy-production:
    if: github.ref_name == 'main'
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22

      - name: Enable Corepack
        run: corepack enable

      - name: Install dependencies
        run: |
          if [ -f pnpm-lock.yaml ]; then
            pnpm install --frozen-lockfile
          elif [ -f yarn.lock ]; then
            yarn install --immutable
          else
            npm ci
          fi

      - name: Build deployment artifact
        run: |
          if [ -f pnpm-lock.yaml ]; then
            pnpm build
          elif [ -f yarn.lock ]; then
            yarn build
          else
            npm run build
          fi

      - name: Install TunnelHub CLI
        run: npm install --global @tunnelhub/cli

      - name: Deploy automation
        env:
          TH_API_HOST: https://api.tunnelhub.io
          TH_CLIENT_ID: ${{ secrets.TH_CLIENT_ID }}
          TH_CLIENT_SECRET: ${{ secrets.TH_CLIENT_SECRET }}
          TH_ENVIRONMENT_ID: ${{ vars.TH_ENVIRONMENT_ID }}
        run: |
          COMMIT_EMAIL="$(git log -1 --pretty=format:'%ae')"
          COMMIT_MESSAGE="$(git log -1 --pretty=format:'%B')"
          COMMIT_SHA="$(git rev-parse --short HEAD)"
          DEPLOY_MESSAGE="${COMMIT_MESSAGE}

          Deploy ${COMMIT_SHA} by ${COMMIT_EMAIL}"

          tunnelhub deploy-automation \
            --environment-id "$TH_ENVIRONMENT_ID" \
            --message "$DEPLOY_MESSAGE"
```

## GitHub Actions Template: develop and main

Use this when `develop` should deploy to `development`, and `main` should deploy to `production`.

```yaml
name: Deploy TunnelHub Automation

on:
  push:
    branches:
      - develop
      - main
  workflow_dispatch:

jobs:
  deploy-development:
    if: github.ref_name == 'develop'
    runs-on: ubuntu-latest
    environment: development

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22

      - name: Enable Corepack
        run: corepack enable

      - name: Install dependencies
        run: |
          if [ -f pnpm-lock.yaml ]; then
            pnpm install --frozen-lockfile
          elif [ -f yarn.lock ]; then
            yarn install --immutable
          else
            npm ci
          fi

      - name: Build deployment artifact
        run: |
          if [ -f pnpm-lock.yaml ]; then
            pnpm build
          elif [ -f yarn.lock ]; then
            yarn build
          else
            npm run build
          fi

      - name: Install TunnelHub CLI
        run: npm install --global @tunnelhub/cli

      - name: Deploy automation
        env:
          TH_API_HOST: https://api.tunnelhub.io
          TH_CLIENT_ID: ${{ secrets.TH_CLIENT_ID }}
          TH_CLIENT_SECRET: ${{ secrets.TH_CLIENT_SECRET }}
          TH_ENVIRONMENT_ID: ${{ vars.TH_ENVIRONMENT_ID }}
        run: |
          COMMIT_EMAIL="$(git log -1 --pretty=format:'%ae')"
          COMMIT_MESSAGE="$(git log -1 --pretty=format:'%B')"
          COMMIT_SHA="$(git rev-parse --short HEAD)"
          DEPLOY_MESSAGE="${COMMIT_MESSAGE}

          Deploy ${COMMIT_SHA} by ${COMMIT_EMAIL}"

          tunnelhub deploy-automation \
            --environment-id "$TH_ENVIRONMENT_ID" \
            --message "$DEPLOY_MESSAGE"

  deploy-production:
    if: github.ref_name == 'main'
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22

      - name: Enable Corepack
        run: corepack enable

      - name: Install dependencies
        run: |
          if [ -f pnpm-lock.yaml ]; then
            pnpm install --frozen-lockfile
          elif [ -f yarn.lock ]; then
            yarn install --immutable
          else
            npm ci
          fi

      - name: Build deployment artifact
        run: |
          if [ -f pnpm-lock.yaml ]; then
            pnpm build
          elif [ -f yarn.lock ]; then
            yarn build
          else
            npm run build
          fi

      - name: Install TunnelHub CLI
        run: npm install --global @tunnelhub/cli

      - name: Deploy automation
        env:
          TH_API_HOST: https://api.tunnelhub.io
          TH_CLIENT_ID: ${{ secrets.TH_CLIENT_ID }}
          TH_CLIENT_SECRET: ${{ secrets.TH_CLIENT_SECRET }}
          TH_ENVIRONMENT_ID: ${{ vars.TH_ENVIRONMENT_ID }}
        run: |
          COMMIT_EMAIL="$(git log -1 --pretty=format:'%ae')"
          COMMIT_MESSAGE="$(git log -1 --pretty=format:'%B')"
          COMMIT_SHA="$(git rev-parse --short HEAD)"
          DEPLOY_MESSAGE="${COMMIT_MESSAGE}

          Deploy ${COMMIT_SHA} by ${COMMIT_EMAIL}"

          tunnelhub deploy-automation \
            --environment-id "$TH_ENVIRONMENT_ID" \
            --message "$DEPLOY_MESSAGE"
```

## Rotation and Revocation

In `Administration > Settings > CI/CD`:

- Use `Regenerar secret` to issue a new secret for an existing credential.
- Use `Revogar` to block new deploys with that credential.

After regenerating a secret, update the GitHub Environment secrets immediately. If migrating to environment-specific credentials, revoke old credentials after the new flow is validated.
