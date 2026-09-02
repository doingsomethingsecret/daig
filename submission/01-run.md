# Phase 1 — Run It & Exit-78 Exercise

## Objective

The objective of Phase 1 was to bring the Daig three-tier application online, verify that the application tiers were healthy and ready, seed the database with reference data, place a real order through the application flow, and understand the deliberately broken Orders container.

## 1. Starting the platform

The application stack was started with:

`make up`

The Makefile maps this to:

`docker compose up -d --build`

The `--build` option ensures that the service images are built from the current Dockerfiles before the containers start. The `-d` option starts the containers in detached mode so that the terminal remains available.

The resulting Compose services were inspected with:

`docker compose ps`

The important distinction is that a container being `Up` only proves that its process is running. It does not by itself prove that the application can serve traffic or that its dependencies are available.

## 2. Health and readiness

The application exposes health and readiness endpoints.

A liveness/health check answers whether the application process itself is alive. It should not depend on the database because a temporary database problem should not cause a healthy application process to be repeatedly restarted.

A readiness check answers whether the service can actually serve traffic. For Orders, readiness performs a database check using `SELECT 1`. If the database is unavailable, readiness returns HTTP 503.

This gives the platform two different signals:

- health/liveness: the process is alive
- readiness: the service is currently capable of serving requests

The distinction prevents a dependency degradation from being converted into a restart loop or broader outage.

## 3. Database seeding

The reference/menu data was initialized with:

`make seed`

The seed operation runs SQL inside the PostgreSQL Compose container. This provides restaurants and menu items required by the application before placing a real order.

## 4. Smoke verification

The stack was then verified with the repository smoke test.

Because this local environment uses custom host port mappings, the smoke test was executed with:

`WEB=http://localhost:18080 ORDERS=http://localhost:13001 KITCHEN=http://localhost:13002 DISPATCH=http://localhost:13003 make smoke`

The custom host ports do not change the internal Docker service ports. They only change how the services are reached from the host machine.

The smoke test verifies that the web tier, Orders, Kitchen, and Dispatch answer successfully and that Orders can reach the database through its readiness check.

## 5. End-to-end order

A real order was created using:

`ORDERS=http://localhost:13001 ./scripts/integration-order.sh`

The script retrieves a restaurant, retrieves a menu item, creates an order, waits briefly for downstream processing, and then checks the final order state.

A successful result is an `ASSIGNED` or `ACCEPTED` state and confirms that the order traversed:

`orders -> kitchen -> dispatch`

This is stronger evidence than individual health checks because it proves that the application tiers can communicate and complete an actual business operation.

## 6. Deliberately broken image

The teaching image was built with:

`make broken`

This builds the `broken` target from:

`services/orders/Dockerfile`

and tags it as:

`tkxel/daig-orders:broken`

The broken image intentionally omits the `DATABASE_URL` configuration.

## 7. Exit code 78

The image was run with:

`docker run --rm tkxel/daig-orders:broken`

The container exits with code `78`.

The shared startup guard defines:

`EX_CONFIG = 78`

This corresponds to a configuration error. When a required environment variable is missing, the guard emits a structured fatal log naming the missing variable and terminates the process with exit code 78.

The important failure message is:

`Missing required configuration: DATABASE_URL`

Therefore, the failure can be diagnosed from the log without guessing: the Orders service was started without its required `DATABASE_URL` configuration.

## Why this failure design is useful

A named configuration error is more useful to an operator than an opaque stack trace. The log identifies the exact missing configuration item and the exit code categorizes the failure as a configuration error.

The operator can immediately determine the likely corrective action: provide the required configuration and restart the service.

## Verified result

Phase 1 demonstrated:

- the Compose stack can be built and started;
- application health and readiness endpoints respond;
- the database can be reached through readiness;
- reference data can be seeded;
- a real order can traverse Orders -> Kitchen -> Dispatch;
- the deliberately broken image fails intentionally;
- the failure identifies `DATABASE_URL` as missing;
- the application uses exit code 78 for configuration errors.

## Evidence

- `submission/phase01/00-environment-baseline.txt`
- `submission/phase01/01-health-readiness-output.txt`
- `submission/phase01/01-smoke-output.txt`
- `submission/phase01/02-seed-output.txt`
- `submission/phase01/03-integration-order-output.txt`
- `submission/phase01/01-broken-build-output.txt`
- `submission/phase01/01-broken-output.txt`
- screenshots in `submission/phase01/screenshots/`
