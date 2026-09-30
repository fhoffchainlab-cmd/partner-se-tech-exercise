# Migration assessment

## Changes and rationale

| Area | Change | Reason |
|---|---|---|
| Base images | Digest-pinned Chainguard Python development and runtime images | Use a minimal runtime and identify the exact tested base images |
| Build structure | Install dependencies in a separate builder; copy a virtual environment without pip | Keep installation tools outside the runtime |
| Python dependencies | Retrieve all packages exclusively from Chainguard Libraries; pin all 13 resolved dependencies | Control the package source and avoid unintended version changes |
| Credentials | Supply a required BuildKit secret from a protected file outside the repository | Keep Libraries credentials out of source files and image layers |
| Web server | Replace the Flask debug server and shell entrypoint with Gunicorn | Use a server suitable for application deployment and support a shell-free runtime |
| Runtime permissions | UID/GID 65532, read-only root filesystem, temporary /tmp, dropped capabilities and no-new-privileges | Reduce runtime write access and operating-system privileges |
| Orchestration | PostgreSQL healthcheck with service_healthy dependency | Wait for database readiness before starting the application |
| Networking | Bind the web port to loopback and remove the published database port | Limit access to the local Docker host and Compose network |
| Development workflow | Remove the source bind mount; retain docker compose up --build | Run the code contained in the built image and rebuild after edits |

Unused requests and humanize dependencies were removed. The obsolete
Compose version field, unused SECRET_KEY example and redundant shell
entrypoint were removed. Git and Docker ignore files exclude local
credentials and Python working files.

The original application failed with:
ModuleNotFoundError: No module named 'psycopg'.

Its dependencies included psycopg2-binary, while the generic PostgreSQL
URL selected a different driver. Using postgresql+psycopg2:// explicitly
aligns the configured driver with the installed package.

## Tradeoffs

- The available public image pair used Python 3.14.7 rather than the
  original Python 3.11. Dependencies were resolved for this runtime and
  the application's CRUD behavior was tested. A customer migration
  would require broader compatibility testing.
- PostgreSQL remains on the original postgres:15 image. The exercise
  focuses on the application container; changing the database major
  version would introduce a separate data migration. Database image
  hardening and digest pinning are follow-up work.
- Image digests and dependency versions improve repeatability but need
  a deliberate update process to receive newer fixes.
- Binary wheels are required. Missing compatible wheels cause the build
  to fail rather than silently use another index or build from source.
- One Gunicorn worker with four threads is sufficient for this small
  local application. Capacity and concurrency need customer-specific
  validation.
- The named database volume and simple startup schema creation were
  retained. Production schema changes should use explicit migrations.
- Default database credentials are for the local exercise. A real
  deployment needs managed credentials and a dedicated application role.
- The web healthcheck checks process availability. A separate readiness
  check should cover database availability in a production deployment.

## Validation performed

- Built the application without cache using Chainguard Libraries.
  The dependency check completed successfully.
- Started the migrated application and tested creating a todo, reloading
  the page, toggling completion and deleting the todo.
- Started a separate Compose project with an empty database volume.
  Both services became healthy; /healthz and / returned HTTP 200.
- Removed the separate test project and restored the normal project.
- Confirmed Python 3.14.7 and runtime UID/GID 65532.
- Inspected the running container: read-only root filesystem,
  CapDrop=["ALL"] and no-new-privileges:true.
- Confirmed pip and /bin/sh were absent from the runtime, and the build
  secret path was absent.
- Confirmed all 13 installed dependencies contained SBOM files.
  This was an inspection of installed metadata, not cryptographic
  verification of package attestations.
- Ran git diff --check.

Testing covered local startup and application behavior. Load testing,
automated vulnerability scanning and signature verification are
follow-up work.

## Next steps in a customer engagement

1. Confirm supported Python versions, deployment targets, database
   requirements and dependency licensing.
2. Establish CI builds with appropriately scoped secret access, automated
   image/dependency updates, vulnerability scanning and attestation
   verification.
3. Add automated integration tests and database readiness monitoring.
4. Introduce schema migrations, managed database credentials, backup and
   restore testing, and an upgrade plan.
5. Assess authentication, authorization and CSRF protection before
   exposing the todo application beyond the local environment.

## AI tool usage

ChatGPT was used throughout the exercise for technical explanations,
researching official documentation, troubleshooting, proposing the
migration design, and generating commands, configuration and draft
documentation.

AI-suggested changes included the multi-stage Dockerfile, dependency
resolution and pins, BuildKit secret handling, Gunicorn configuration,
Compose hardening and the explicit PostgreSQL driver selection.

I executed the commands in my own VM, inspected build and runtime output,
tested application behavior in the browser, and checked the proposed
settings against the running container. Acceptance was based on these
observed results and the reasons documented above. The configuration and
documentation were AI-assisted; the migration design is not presented
as independently authored.

## References

- https://images.chainguard.dev/directory/image/python/overview
- https://edu.chainguard.dev/chainguard/libraries/python/
- https://docs.docker.com/build/building/secrets/
- https://docs.docker.com/compose/how-tos/startup-order/
