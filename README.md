# Partner SE Technical Exercise Starter Repo

This repository contains a small Flask + Postgres todo application intended for use as a Partner SE technical exercise.

## Scenario

A customer has a small Python application that runs locally with Docker Compose. It works, but the application container and dependency setup are not aligned with their supply chain and container hardening goals.

The exercise for the candidate is to improve this application by:

- migrating the application container to Chainguard Containers
- configuring Python dependency installation to use Chainguard Libraries only
- preserving a simple local developer workflow with Docker Compose
- making practical, high-impact improvements without over-engineering the solution

## Timebox

- Expected: 90 minutes
- Max: 2 hours

## Starter repo notes

This starter intentionally includes several issues for the candidate to evaluate and improve. Examples include container image choice, build structure, dependency installation, runtime posture, and local orchestration behavior.

## Run locally

```bash
cp .env.example .env   # then set real values
docker compose up --build
```

Then open <http://localhost:8000>.

The build installs Python packages from Chainguard Libraries and needs credentials in `~/.netrc`. See
[WRITEUP.md](WRITEUP.md#running-it-locally) for the setup and for building with Podman.

## ⚠️ Chainguard Libraries Access

**Configuring Python dependency installation to use Chainguard Libraries is a required part of this exercise.**

If you have not been provided with a Chainguard Libraries token and identity ID, **please request one before starting**. You will need these credentials to complete the Libraries portion of the exercise. Do not submit without attempting this step — if access cannot be arranged, document what you would have done and why.

## Candidate deliverables

Candidates should submit a link to their forked repository containing:

1. Updated code and configuration
2. A write-up **committed to the repository** (Markdown preferred) describing:
   - what they changed
   - why they changed it
   - what risks or issues were reduced
   - tradeoffs they made
   - what they would do next in a real customer engagement
3. A brief note on AI tool usage — specifically: which tools you used, where in the exercise you used them, and how you distinguished AI-suggested changes from your own judgment. There is no penalty for using AI; transparency about how you used it is what matters.

## Interviewer notes

A strong solution will usually:

- use appropriate Chainguard Python image variants
- separate build and runtime concerns sensibly
- remove unnecessary packages and dependencies
- avoid running as root
- keep `docker compose up --build` working
- clearly explain prioritization and tradeoffs
