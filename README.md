# FestPass: Docker & Container Security Assessment (CA-2)

Containerised three-tier registration app (nginx frontend, Flask backend, PostgreSQL), scanned with Trivy,
published to Docker Hub with semantic version tags and signed with cosign.

## Student details

| | |
|---|---|
| **Name** | Naman Kansal |
| **REGNO** | 202401100100160 |
| **PORT** | 8060 |
| **Docker Hub username** | `namankansal23` |
| **Docker Hub repository** | <https://hub.docker.com/r/namankansal23/fest-backend> (public) |
| **GitHub repository** | <https://github.com/NamanKansal230505/202401100100160-docker-assessment> |

## Released image

| Tag | Digest | Signed |
|---|---|---|
| `1.0.0` | `sha256:dd8fcda1e1fd111d100744781434ea77dceb74b5f27a5976a73b79cb1d42cfcb` | Yes, cosign key pair, signed by digest |
| `1.0` | same as `1.0.0` | (same digest) |
| `1.0.1` | `sha256:45943f84b9e48e67514a772f8a58aec22dd375a74e092f97c646ceb14168dc11` | **No**: `cosign verify` rejects it |

**1.0.0 digest as pushed to Docker Hub:** `sha256:dd8fcda1e1fd111d100744781434ea77dceb74b5f27a5976a73b79cb1d42cfcb`.
This is the OCI image index that `docker push` reported. The Docker Hub Tags page lists the linux/amd64 manifest inside it,
`sha256:125ac0c5991192edeb5b5ed81d5cfb8f517c4fa9ba88e10b44b5d20052d97540`.

## Vulnerability scan: before and after (Stage 3)

Scanner: Trivy 0.76.1 (`trivy image --scanners vuln`), DB updated 2026-10-28.
Full reports: [`reports/trivy-before.txt`](reports/trivy-before.txt), [`reports/trivy-after.txt`](reports/trivy-after.txt).

| Image | Base image | CRITICAL | HIGH | Size (content / on disk) |
|---|---|---:|---:|---:|
| `fest-backend:0.1` | `python:3.9` (Debian 13.1, full) | **234** | **2249** | 410 MB / 1.6 GB |
| `fest-backend:0.2` | `python:3.14.10-slim-trixie` (Debian 13.7) | **0** | **44** | 49.8 MB / 203 MB |

`trivy image --severity CRITICAL --ignore-unfixed --exit-code 1 fest-backend:0.2` exits with **0** (S3-3).
With `--ignore-unfixed`, 0.2 has no fixable findings at any severity. The 44 HIGH left are Debian packages with no upstream fix yet.

What changed between 0.1 and 0.2:

- **Base image:** `python:3.9` (full Debian, Python 3.9 is end-of-life) changed to `python:3.14.10-slim-trixie`, pinned to an exact release, never `latest`.
- **Libraries:** Flask 2.2.2 became 3.1.3, Werkzeug 2.2.2 became 3.1.9, and psycopg2-binary 2.9.10 became 2.9.13. Flask's own dependencies are pinned too.
- **Multi-stage build:** dependencies are installed in a build stage. The runtime image has **no pip**, whose vendored `urllib3`/`msgpack` carried HIGH CVEs and which is not needed to run the app.
- **OS fixes:** `apt-get upgrade` applies Debian security fixes released after the base image was published.
- **Non-root user:** the app runs as an unprivileged system user, `festpass` (uid/gid 10001).
- **Healthcheck and labels:** a `HEALTHCHECK` is built into the image, along with OCI labels that include the release version.

## Architecture (Stage 2)

```
browser --:8060--> frontend (nginx:1.30-alpine) --frontend-net--> backend (Flask :5000) --backend-net--> db (postgres:17-alpine)
                                                                                         (internal)      volume: db-data
```

- **Networks:** only `backend` is on both networks, so the frontend cannot resolve or reach `db` (S2-2: `ping: bad address 'db'`).
  `backend-net` is `internal: true`, so the database has no route outside the stack.
- **Healthcheck:** `db` has a `pg_isready` healthcheck, and `backend` starts only once it passes.
- **Persistence:** data lives in the named volume `festpass_db-data`. Registrations survive `docker compose down` / `up` (S2-4).
- **Secrets:** the database password comes from `.env`, which is git-ignored. [`.env.example`](.env.example) is the committed template.

## Run it

```powershell
cp .env.example .env          # then set DB_PASSWORD
docker compose up -d          # pulls namankansal23/fest-backend:1.0.0 from Docker Hub
start http://localhost:8060
cosign verify --key cosign.pub namankansal23/fest-backend:1.0.0
```

## Evidence

| Task | Evidence |
|---|---|
| 1.1 | [`backend/Dockerfile`](backend/Dockerfile) (Stage 1 version in commit "Stage 1 done"), [`backend/.dockerignore`](backend/.dockerignore) |
| 1.2 | [S1-2](screenshots/S1-2.png): `fest-backend:0.1` built and listed with its size |
| 1.3 | [S1-3](screenshots/S1-3.png): `docker ps`, `/api/health` with REGNO, logs, container removed |
| 2.1 | [`compose.yaml`](compose.yaml) (Stage 2 version in commit "Stage 2 done"), [`.env.example`](.env.example) |
| 2.2 | [S2-2](screenshots/S2-2.png): frontend cannot ping `db`, backend can resolve it |
| 2.3 | [S2-3](screenshots/S2-3.png): two registrations added in the browser, page shows REGNO |
| 2.4 | [S2-4](screenshots/S2-4.png): down/up keeps both registrations, `docker volume ls` |
| 3.1 | [S3-1](screenshots/S3-1.png), [`reports/trivy-before.txt`](reports/trivy-before.txt) |
| 3.2 | [`backend/Dockerfile`](backend/Dockerfile), [`backend/requirements.txt`](backend/requirements.txt) |
| 3.3 | [S3-3](screenshots/S3-3.png), [`reports/trivy-after.txt`](reports/trivy-after.txt), table above |
| 4.1 | [S4-1](screenshots/S4-1.png): build with `APP_VERSION=1.0.0`, push of `1.0.0` and `1.0` with digest |
| 4.2 | [S4-2](screenshots/S4-2.png): Docker Hub Tags page; digest recorded above |
| 4.3 | [`compose.yaml`](compose.yaml), [S4-3](screenshots/S4-3.png) (page shows backend 1.0.0), [S4-3-terminal](screenshots/S4-3-terminal.png) (local images deleted, pulled from Docker Hub) |
| 5.1 | [S5-1](screenshots/S5-1.png), [`cosign.pub`](cosign.pub); `*.key` is git-ignored and `cosign.key` is in no commit |
| 5.2 / 5.3 | [S5-2_S5-3](screenshots/S5-2_S5-3.png): signed by digest, verified with `cosign.pub` |
| 5.4 | [S5-4](screenshots/S5-4.png): unsigned `1.0.1` rejected (`no signatures found`, exit 10) |

## AI tools used

- **Claude (Anthropic):** writing the Dockerfiles, `compose.yaml` and README

## Steps not completed

- None. All tasks in Stages 1–5 were completed.
