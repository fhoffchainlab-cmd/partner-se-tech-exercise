# Chainguard Python Todo Application

A Flask + PostgreSQL todo application migrated to Chainguard Python
Containers and Chainguard Libraries.

The original assessment is preserved in [EXERCISE.md](EXERCISE.md).
Migration decisions, validation and AI usage are documented in
[ASSESSMENT.md](ASSESSMENT.md).

## Requirements

- Linux containers with Docker Engine and Docker Compose.
- Valid Chainguard Libraries Python identity ID and token.
- The application was tested on an Ubuntu 24.04 AMD64 Hyper-V VM.

## Configure Libraries access

Create a credentials file outside the repository. Run the following in
Bash, entering each credential at its prompt:

```bash
mkdir -p "$HOME/.config/chainguard-lab"
read -r -p "Identity ID: " CHAINGUARD_PYTHON_IDENTITY_ID
read -r -s -p "Token: " CHAINGUARD_PYTHON_TOKEN
printf '\n'
(
  umask 077
  printf 'machine libraries.cgr.dev\nlogin %s\npassword %s\n' \
    "$CHAINGUARD_PYTHON_IDENTITY_ID" \
    "$CHAINGUARD_PYTHON_TOKEN" \
    > "$HOME/.config/chainguard-lab/python.netrc"
)
chmod 600 "$HOME/.config/chainguard-lab/python.netrc"
unset CHAINGUARD_PYTHON_IDENTITY_ID CHAINGUARD_PYTHON_TOKEN
```

For another file location, set `CHAINGUARD_NETRC_PATH` to its absolute path.

Compose supplies this file as a BuildKit secret during dependency
installation. It is not mounted into the running application.
The build uses only `https://libraries.cgr.dev/python/simple/`, with no
additional Python package index.

## Run locally

```bash
docker compose up --build
```

Open http://localhost:8000 on the Docker host.

To start in the background and wait for healthchecks:

```bash
docker compose up --build -d --wait --wait-timeout 90
docker compose ps
curl -i http://localhost:8000/healthz
```

For a remote VM, run this on your workstation, replacing the user and host:

```bash
ssh -N -L 127.0.0.1:18000:127.0.0.1:8000 YOUR_USER@YOUR_VM_IP
```

Keep the SSH connection open and browse to http://localhost:18000.

## Configuration and persistence

The default PostgreSQL credentials are for this local exercise.
PostgreSQL is reachable within the Compose network; its port is not
published to the host. The web port binds to host loopback.

`DATABASE_URL` can be overridden through the environment or a local
`.env` file. `.env.example` documents the default connection URL.
An override must match the database's actual configuration.

Todo data persists in the named `postgres_data` volume.

```bash
docker compose down
```

This stops the application while retaining the database volume.

## Troubleshooting

```bash
docker compose logs --tail=100 web db
docker compose config --quiet
docker compose --progress plain build --no-cache web
```

A missing or expired Libraries credential must be corrected before a
fresh dependency installation can succeed. Build secrets do not
automatically invalidate cached build steps; use `--no-cache` to test
package retrieval again.

The `/healthz` endpoint checks the web process. Loading `/` also exercises
a database query.
