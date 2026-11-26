# ETL GitOps: Application Repo

A small Python ETL job wired into a **GitOps delivery pipeline**. CI tests the code, builds a container image tagged with the commit SHA, pushes it to Azure Container Registry, and then commits the new tag into a separate **config repo**. Argo CD watches that config repo and deploys it to Kubernetes.

This is the application half. The deployment half is [`etl-gitops-config`](https://github.com/burhanuddin-123/etl-gitops-config).

## Pipeline

```
 development branch / PR     main branch
 ───────────────────────     ─────────────────────────────────────────────
 pytest (unit + integ.)      build image ──► push to ACR (tag = commit SHA)
 docker build (PRs)                                  │
                                       commit new tag into etl-gitops-config
                                                     │
                                  Argo CD syncs the cluster from the config repo
```

| Trigger | What runs |
|---|---|
| Pull request to `main`, or push to `development` | `pytest tests/`; PRs also do a `docker build` to catch Dockerfile breakage |
| Push to `main` | Build and push the image, then update `app/etl/dev/cronjob.yaml` in the config repo |

Workflow: [`.github/workflows/ci-cd.yml`](.github/workflows/ci-cd.yml)

## The ETL job

`etl_script.py` extracts users from the public JSONPlaceholder API, transforms them with pandas (renames columns, flattens the nested company field) and loads the result to `output/processed_users.csv`. It is deliberately simple. The point of this repo is the delivery pipeline around it.

## Run it locally

```bash
pip install -r requirements.txt
pytest tests/            # 1 unit test + 1 integration test (needs internet)
python etl_script.py     # writes output/processed_users.csv
docker build -t etl-app . && docker run --rm etl-app
```

## Required GitHub Actions secrets

| Secret | Purpose |
|---|---|
| `ACR_LOGIN_SERVER` | Registry host, e.g. `myregistry.azurecr.io` |
| `ACR_USERNAME` / `ACR_PASSWORD` | Registry credentials |
| `CONFIG_REPO_PAT` | Token with write access to the config repo |

## Notes

- Images are tagged with the full commit SHA, so every deploy is traceable, and a rollback is a revert in the config repo.
- I cleaned up and re-published this repo recently. The tests above pass, but I haven't re-run the cloud side (ACR, Argo CD) since the cleanup.

## License

MIT, see [LICENSE](LICENSE).
