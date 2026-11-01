# Chainguard migration: write-up

The todo app now builds and runs entirely on Chainguard: a multi-stage `cgr.dev/chainguard/python`
image (non-root, no shell in the runtime image), `cgr.dev/chainguard/postgres`, and Python dependencies
installed only from Chainguard Libraries. `docker compose up --build` is still the workflow.

| Image | Before (grype) | After (grype) |
|---|---|---|
| App | 436 findings (16 Critical, 182 High) | **3** (1 Critical, 0 High), all in the base Python, no fix released yet |
| Postgres | 313 findings (1 Critical, 89 High) | **0** |

## How I approached it

1. **Static analysis first (about an hour).** I read every file in the repo before running anything
   and wrote down what I'd expect to change. I used a voice assistant for quick background questions,
   such as what `netcat-traditional` is for and why `rm -rf /var/lib/apt/lists/*` is used.
2. **Got the starter running as-is.** It didn't start (see below), so I fixed that first to have a known-good baseline.
3. **Chainguard Libraries and Postgres**, then **committed a working checkpoint** while the app was still on
   `python:3.11-slim`, before taking on a large Python version jump.
4. **Chainguard Python**, then hardening, a vulnerability scan and dependency fixes, each as its own commit and tested.

## What I found in the starter

- **It didn't run.** `requirements.txt` pins the top-level packages, but not their dependencies. `SQLAlchemy`
  (pulled in by Flask-SQLAlchemy) resolved to 2.1, which switched the default Postgres driver to psycopg 3,
  so the app crashed with `No module named 'psycopg'`. This is a good example of why top-level pins aren't
  enough: you need the full dependency tree pinned, plus an SBOM and scanning, not just a scan of the top-level packages.
- **Credentials hardcoded in three places:** `docker-compose.yml`, `.env.example`, and a fallback connection
  string in `app.py`. If the environment variable were missing, the app would silently connect using the fallback.
- **Flask dev server with `debug=True`** as the entrypoint, plus `FLASK_ENV=development`.
- **`curl`, `vim`, `netcat`** installed in the image. They're useful for troubleshooting, but whether to include them is an ops decision, and they add attack surface.
- **Runs as root**, on a Debian-based `python:3.11-slim`.
- **Unused dependencies:** the app never imports `requests` or `humanize`.
- Out of scope, but worth raising with a customer: the app has no authentication or authorisation, so anyone
  can create or delete todos. SQL injection is handled by SQLAlchemy's ORM using parameterised queries.

## Running it locally

```bash
cp .env.example .env          # then set real values
docker compose up --build
```

Python dependencies need Chainguard Libraries credentials in `~/.netrc`:

```
machine libraries.cgr.dev
login <identity id>
password <token>
```

**Podman on macOS (what I used):** compose through Podman's Docker-compatible API doesn't pass build
secrets, so I build the image directly and then start compose without building:

```bash
podman build --secret id=netrc,src=$HOME/.netrc -t chainguard_explore-web ./app
podman tag localhost/chainguard_explore-web:latest docker.io/library/chainguard_explore-web:latest
docker compose up --no-build
```

Podman on macOS also leaves a plaintext copy of the secret (`podman-build-secret-*`) next to the build
context. It's git-ignored here. Building inside the VM (`podman machine ssh -- podman build ...`) avoids it.
I haven't tested `docker compose up --build` on Docker Desktop.

## Remaining issues and next steps

In this repo:
- **3 findings in the base Python 3.15 package**, with no fix released yet. Only one is relevant: the `ssl` module's
  list of known-compromised certificates. It only matters when Python makes outbound TLS connections, and this app makes none.
  Fix: rebuild when Chainguard ships a patched image. Meanwhile, record the assessment in a `.grype.yaml` ignore rule or a VEX statement, rather than hiding the finding.
- Move the `./app:/app` bind mount into a dev-only override file. As it stands, it replaces the image's code and lets the
  app write to its own source.
- Remove the Postgres `5432` port mapping (nothing outside compose needs it). Consider `user: postgres` so the
  database container never starts as root (its entrypoint currently starts as root and then drops to the `postgres` user).
- Add `USER 65532` explicitly to the runtime stage, so non-root doesn't depend on the base image.
- `SECRET_KEY` is passed to the app but `app.py` never reads it.

In a customer engagement:
- I'd need to know where the app is hosted before going further on separating build and runtime, and on secrets.
- Use the customer's organisation for version-pinned images, pinned by digest and updated automatically (Renovate or Dependabot).
- Lock the full dependency tree with hashes (`pip-compile --generate-hashes` or `uv lock`).
- Run grype in CI and fail on High or Critical findings. Verify image signatures and SBOMs (`cosign verify`).
- Short-lived pull tokens in CI, and secrets from the platform (Kubernetes Secrets with encryption at rest, or Vault).

## Questions for the reviewers

- Was I meant to have registry access (an organisation with version-tagged images)? I only had a
  Libraries token, which can't pull images. If I missed how to use it, I'd like to know.
- Is moving to the latest Python and Postgres acceptable here, given that limitation? I'm happy to redo this
  with the same versions as the starter (3.11 / 15) if I can get access to an organisation that has them.

## Feedback on the Chainguard onboarding

- It wasn't obvious that the Libraries token gives package access only, not registry access.
- Creating an organisation needs a company email domain; a personal email didn't work.
- `chainctl images discover` returned "no organizations found" even after I created a starter organisation.
- The container-variants page could link to a list of what each `-dev` image adds (or its SBOM).

## What I changed and why

### Application image

| Change | Why |
|---|---|
| Multi-stage build: `python:latest-dev` builds a virtualenv, and `python:latest` runs it | Build tools (shell, `apk`, compiler) stay in the build stage. The runtime image has no shell or package manager and runs as `nonroot` (uid 65532, the base image's built-in user). |
| `pip install --index-url https://libraries.cgr.dev/python/simple/` | Dependencies come only from Chainguard Libraries. `--index-url` (not `--extra-index-url`) means there is no fallback to PyPI, which also prevents dependency confusion. |
| Libraries credentials passed as a build secret (`RUN --mount=type=secret,id=netrc`) | The `.netrc` exists only during the `pip install` step and never ends up in an image layer or the build history. |
| `gunicorn` instead of the Flask debug server; removed `entrypoint.sh` | The Werkzeug debugger allows code execution if exposed. There's no shell in the runtime image, so gunicorn runs directly from `ENTRYPOINT`. |
| Removed `curl`, `vim`, `netcat` | Not needed at runtime. The `-dev` image is available for debugging. |
| `apk add postgresql-18-dev` in the build stage; `libpq` and its shared libraries copied into the runtime image | On Python 3.15 (see Tradeoffs), `psycopg2` has to be compiled from source. I generated the list of libraries from the compiled module's actual dependencies. |
| Disabled gunicorn's control socket | gunicorn 25.1+ opens a management socket by default, which this app doesn't need. |
| Removed `# syntax=docker/dockerfile:1` | On Docker it downloads the Dockerfile parser from Docker Hub, the only non-Chainguard component in the build. Docker 23+ supports build secrets without it. |
| Added `.dockerignore` | Keeps `__pycache__` and the Dockerfile out of the image. |

### Dependencies

| Change | Why |
|---|---|
| Pinned `SQLAlchemy==2.0.36` | Fixes the startup crash described above. |
| `gunicorn` 21.2.0 → 26.2.0, `flask` 2.3.3 → 3.1.3 | grype found 2 High (request smuggling) and 1 Low finding. |
| Removed `requests` and `humanize` | Not used by the app. `requests` also had 3 Medium findings. |
| `psycopg2-binary` 2.9.9 → 2.9.11 | A version Chainguard Libraries provides. |

### Database, compose and secrets

| Change | Why |
|---|---|
| `postgres:15` → `cgr.dev/chainguard/postgres:latest` | Minimal image; 0 findings. |
| `init.sql` mounted at `/var/lib/postgres/initdb/` | The Chainguard Postgres image reads init scripts from this path. The Docker Hub path (`/docker-entrypoint-initdb.d/`) is **silently ignored**, a real difference a customer would hit when migrating. |
| `pg_isready` healthcheck + `depends_on: condition: service_healthy` | The app connects at startup and was crashing before Postgres was ready. |
| Credentials moved to `.env` (git-ignored), with `.env.example` as the template; no fallback in `app.py` | No secrets in tracked files. Compose refuses to start with a clear message if a value is missing. In a real deployment these should come from the platform (Kubernetes Secrets, Vault or a cloud secrets manager), not a file. |
| `.gitignore` for `.netrc`, `.env` and `podman-build-secret-*` | See the Podman note above: Podman was writing copies of the credentials into the project folder. |

## Tradeoffs

- **Python 3.11 → 3.15 and Postgres 15 → 18.** The free Chainguard tier only provides `latest` and
  `latest-dev`. Version tags like `python:3.11` and `postgres:15` need a paid organisation. The Libraries
  token I was given works for Python packages but returns `403` from the image registry. So I committed a working
  3.11 checkpoint first, then moved to `latest`. For a customer I'd use their organisation's
  `3.11` and `15` tags and treat any version upgrade as a separate, planned change with proper testing.
- **Compiling C extensions.** Chainguard Libraries doesn't yet have prebuilt Python 3.15 versions of
  `psycopg2` or `MarkupSafe`, so they compile in the build stage, and `libpq` plus Kerberos, LDAP and SASL
  libraries are copied into the runtime image. That list is the most fragile part of the Dockerfile. It
  wouldn't be needed on `3.11`. A pure-Python driver such as `pg8000` would remove it too, but that changes the app's driver.
- **`latest` tags, not digests.** This keeps the exercise simple, but builds aren't reproducible. Production should pin by digest.
- **Upgraded Flask to a new major version (2 → 3).** It fixes a vulnerability and the app only uses basic Flask features. I tested adding,
  completing and deleting todos, static files and 404s.

## AI tool usage

- **Analysis:** done by reading the code myself, with quick background questions to Google Assistant / Gemini.
- **Chainguard access and finding registries:** Chainguard's website AI assistant.
- **Claude Code (in VS Code):** diagnosing the startup crash and the Postgres connectivity issues, Podman build
  secrets, getting packages to work without Python 3.11 images, the non-root changes, the grype scans and the dependency upgrades. It also drafted this
  write-up from my notes, and I reviewed and edited it.
- **How I kept my own judgement in the loop:** I ran the commands, read the explanations and questioned changes
  before accepting them. For example, I asked how the non-root user was set up and where 65532 came from, and I asked it to check that every package came from
  Chainguard. Claude removed the unused packages before I got to it. I'd already identified them in my static
  analysis, so I checked the result by testing the app instead of redoing it. Every commit was built and tested locally.

A PDF of my running notebook for the process can be shared to show thinking, dead ends and attempts.