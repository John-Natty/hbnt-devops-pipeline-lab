
# Deployment Runbook

## Trigger

The deployment pipeline runs on a push to the `main` branch.

The jobs run in this order:

```text
test -> build -> deploy
```

The `build` job depends on `test`, and the `deploy` job depends on `build`.

Pull request runs can execute the test job, but they do not publish or deploy an image.

## Staging target

The staging environment is hosted on Render.

The application runs as a Render Web Service using the Docker image published to GitHub Container Registry.

Each deployment uses the image associated with the current Git commit SHA:

```text
ghcr.io/john-natty/hbnt-devops-pipeline-lab:<commit-sha>
```

The `latest` tag is also published for convenience.

## Database

The staging application uses a disposable PostgreSQL database hosted on Render.

The Web Service receives the database connection string through the `DATABASE_URL` environment variable.

The internal Render PostgreSQL connection URL is used so that the Web Service can communicate with the database through Render's private network.

Database credentials and connection strings must not be committed to the repository.

## Verification

After triggering the Render deployment, the GitHub Actions workflow verifies the staging service.

Two endpoints are checked independently:

```text
GET /health -> HTTP 200
GET /items  -> HTTP 200
```

`/health` confirms that the application process is running.

`/items` confirms that the deployed API can communicate with PostgreSQL.

The workflow performs a maximum of 20 verification attempts with 15 seconds between attempts.

The deployment succeeds only when both endpoints return HTTP 200.

If both checks do not succeed within the allowed attempts, the deploy job exits with a non-zero status and the pipeline fails.

## Rollback

If a staging deployment is faulty, an earlier known-good Docker image can be redeployed using its immutable Git commit SHA tag.

Example:

```text
ghcr.io/john-natty/hbnt-devops-pipeline-lab:<previous-commit-sha>
```

Using the SHA tag makes it possible to identify and redeploy the exact image produced from a specific Git commit instead of relying on the moving `latest` tag.

## Credentials

Deployment credentials are stored in GitHub repository secrets.

The workflow uses:

```text
RENDER_API_KEY
RENDER_SERVICE_ID
```

The staging URL is stored as the GitHub repository variable:

```text
STAGING_URL
```

No credential value is stored directly in the workflow, repository, logs, or this runbook.

## Cleanup

When the lab is no longer needed:

1. Remove the Render staging Web Service.
2. Remove the temporary Render PostgreSQL database.
3. Revoke or delete the temporary Render API key.
4. Remove `RENDER_API_KEY` and `RENDER_SERVICE_ID` from GitHub repository secrets if they are no longer needed.
5. Remove the `STAGING_URL` repository variable if it is no longer needed.
6. Keep the GHCR package visibility consistent with the intended cleanup policy.
