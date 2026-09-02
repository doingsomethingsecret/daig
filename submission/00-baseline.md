# Phase 0 — Environment & Baseline

## Objective

The objective of Phase 0 is to establish a clean starting point, verify the local development/DevOps tooling, install the repository's local quality gate, and confirm that the repository passes its static checks before runtime work begins.

## Environment Verification

The project was verified from the local repository with the following tools:

- Git
- Docker
- Docker Compose
- GNU Make

The Docker Compose configuration was also parsed successfully with:

`docker compose config -q`

This verifies that the Compose configuration can be parsed by Docker Compose, but it does not prove that the application containers are running or that the application is functionally healthy.

## Local Quality Gate

The repository's Git hooks were installed using:

`make hooks`

The local static verification gate was then executed using:

`make check`

The purpose of running the local gate before starting the application is to detect syntax, formatting, structural, and repository-level issues early, before spending time debugging runtime behaviour.

## Verified vs Unverified

The Phase 0 checks provide static verification only. They verify repository and configuration properties that can be checked without starting the full application stack. They do not prove that Docker containers can start successfully, that services can communicate with one another, that PostgreSQL or Redis are reachable, that health/readiness endpoints work, or that a real end-to-end order can be completed. Those behaviours require runtime and integration verification in later phases.

This distinction matters because a static check can pass while the deployed application still fails due to runtime configuration, networking, dependencies, credentials, database state, or application behaviour. Therefore, the result of `make check` should be treated as one verification layer rather than proof that the entire platform works.

## Result

Phase 0 establishes the baseline quality gate before runtime experimentation begins.

Evidence:

- `submission/phase01/00-environment-baseline.txt`
- `submission/phase01/screenshots/00-environment-baseline.png`

