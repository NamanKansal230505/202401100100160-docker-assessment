# FestPass: Docker & Container Security Assessment (CA-2)

| | |
|---|---|
| **Name** | Naman Kansal |
| **REGNO** | 202401100100160 |
| **PORT** | 8060 |

## Stage 3: Trivy scan, before and after

Scanned with Trivy 0.76.1 (`trivy image --scanners vuln`). Full reports: [`reports/trivy-before.txt`](reports/trivy-before.txt), [`reports/trivy-after.txt`](reports/trivy-after.txt).

| Image | Base | CRITICAL | HIGH | Image size (content / on disk) |
|---|---|---:|---:|---:|
| `fest-backend:0.1` | `python:3.9` (Debian 13.1, full) | **234** | **2249** | 410 MB / 1.6 GB |
| `fest-backend:0.2` | `python:3.14.10-slim-trixie` (Debian 13.7) | **0** | **44** | 49.8 MB / 203 MB |

`trivy image --severity CRITICAL --ignore-unfixed --exit-code 1 fest-backend:0.2` exits with **0** (screenshot S3-3).
With `--ignore-unfixed`, 0.2 has no fixable findings at any severity: the 44 HIGH left are Debian packages with no upstream fix yet.
