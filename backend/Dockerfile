# FestPass backend - hardened image.
# Current Python on Debian 13 slim, pinned to an exact release (never "latest").

# ---- build stage: install the pinned dependencies into /install ----
FROM python:3.14.10-slim-trixie AS build
ENV PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1
# Copied on its own so this layer is cached until requirements.txt changes.
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

# ---- runtime stage: no pip, no build tools, non-root ----
FROM python:3.14.10-slim-trixie

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Pick up Debian security fixes published after the base image was built, drop pip
# (its vendored urllib3/msgpack carry HIGH CVEs and it is not needed at runtime),
# and create an unprivileged system user to run the app.
RUN apt-get update \
 && apt-get upgrade -y --no-install-recommends \
 && rm -rf /var/lib/apt/lists/* \
 && python -m pip uninstall -y pip \
 && rm -rf /usr/local/lib/python3.14/ensurepip \
 && groupadd --system --gid 10001 festpass \
 && useradd --system --uid 10001 --gid festpass --no-create-home --shell /usr/sbin/nologin festpass

COPY --from=build /install /usr/local

WORKDIR /app
# Application code changes most often, so it is copied last.
COPY app.py .

# Release version, set at build time: docker build --build-arg APP_VERSION=1.0.0 ...
# Declared after the slow layers so a version bump does not invalidate them.
ARG APP_VERSION=dev
ENV APP_VERSION=${APP_VERSION}
LABEL org.opencontainers.image.title="fest-backend" \
      org.opencontainers.image.description="FestPass registration API" \
      org.opencontainers.image.version="${APP_VERSION}" \
      org.opencontainers.image.source="https://github.com/NamanKansal230505/202401100100160-docker-assessment"

USER 10001:10001

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:5000/api/health', timeout=2)"]

CMD ["python", "app.py"]
