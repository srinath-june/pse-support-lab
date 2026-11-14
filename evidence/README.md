# Evidence checklist (fill after running `make up` and `make load`)

- incident1: Grafana import-duration screenshot; `docker compose logs api | grep import`; psql output (imports + OBS-2611 invoices)
- incident2: `docker inspect <api> --format '{{.State.OOMKilled}} {{.RestartCount}}'`; api memory panel; Loki "api starting" lines; stuck Validated query
- incident3: psql output for ANF-2611-% invoices; ASP 422 BR-CO-15 response
- incident4: Loki asp-mock "rate limit exceeded"; api status_code=429; psql status counts; curl ASP /v1/documents?sender=...
- incident5: portal screenshot of AE-BR-04 error
