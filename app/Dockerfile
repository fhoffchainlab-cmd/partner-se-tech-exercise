# syntax=docker/dockerfile:1

FROM cgr.dev/chainguard/python:latest-dev@sha256:316dcb52b594ca1b54557cfe41fa66efb4e5bb61ef8b5f262157fbb614cb5f9d AS builder

USER root
WORKDIR /build

COPY requirements.txt .
RUN python -m venv --without-pip /opt/venv

RUN --mount=type=secret,id=chainguard_netrc,target=/run/chainguard.netrc,required=true \
    NETRC=/run/chainguard.netrc \
    PIP_CONFIG_FILE=/dev/null \
    PIP_EXTRA_INDEX_URL= \
    python -m pip --python /opt/venv/bin/python install \
    --no-cache-dir --disable-pip-version-check --no-input \
    --only-binary=:all: \
    --index-url https://libraries.cgr.dev/python/simple/ \
    -r requirements.txt

RUN python -m pip --python /opt/venv/bin/python check

FROM cgr.dev/chainguard/python:latest@sha256:38ba1cbf71702bacc5f5be22ea41e3d4ad1bfb2565413b0caf4bafde38e831f2 AS runtime

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/opt/venv/bin:$PATH"

COPY --from=builder /opt/venv /opt/venv
COPY app.py .
COPY templates ./templates
COPY static ./static

USER 65532:65532

EXPOSE 8000

ENTRYPOINT ["/opt/venv/bin/python", "-m", "gunicorn"]
CMD ["--bind", "0.0.0.0:8000", "--workers", "1", "--threads", "4", "--worker-tmp-dir", "/tmp", "--access-logfile", "-", "--error-logfile", "-", "app:app"]
