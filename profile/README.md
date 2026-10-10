# GLOW

A food delivery platform in the style of Wolt, built by students at Aarhus University as part of an R&D project, designed as our own custom course, Backend Architecture and Scalability. The point of the project is to learn how a real backend system is split into services, containerised, deployed and kept running.

GLOW is made up of seven Java microservices, a React frontend and a shared DevOps repository. Each service has its own database and ships as its own Docker image, users sign in through Keycloak, and the whole system is deployed to Kubernetes with Helm by an automated CI/CD pipeline.

> **Live deployment:** GLOW runs on the university's Kubernetes cluster at [project.orbit.au.dk/alt-2026f01](https://project.orbit.au.dk/alt-2026f01/), but the site is only reachable from the Aarhus University network.
>
> GLOW is an educational project and is not affiliated with Wolt. Payments are simulated and no real money is involved.

## What you can do in GLOW

- **Customers** browse restaurants and their menus, fill a cart, check out and pay.
- **Restaurants** receive incoming orders and update them as they are prepared.
- **Couriers** have profiles, can set their availability and receive orders to deliver (in progress).

## Architecture

```text
                 ┌──────────────────────────┐
                 │   glow-ui  (React SPA)   │
                 └────────────┬─────────────┘
                              │
                 ┌────────────▼─────────────┐
                 │       Edge router        │   Traefik locally,
                 │  /   ·  /auth  ·  /api/* │   Envoy Gateway in production
                 └──────┬─────────────┬─────┘
                        │             │
              ┌─────────▼──┐   ┌──────▼───────────────────────────┐
              │  Keycloak  │   │       7 Quarkus services         │
              │   (OIDC)   │   │ user · restaurant · menu · cart  │
              └────────────┘   │    order · payment · courier     │
                               └──────┬───────────────────┬───────┘
                                      │                   │
                         ┌────────────▼────────┐   ┌──────▼───────┐
                         │     PostgreSQL      │   │ stripe-mock  │
                         │ one DB per service  │   │              │
                         └─────────────────────┘   └──────────────┘
```

The services talk to each other over REST. When a customer checks out, the order service asks the payment service for a payment intent, marks the order as processing and notifies the restaurant. The restaurant then sends status updates back as the order moves from preparing to delivering to completed. Payments go through the official Stripe Java SDK, pointed at [stripe-mock](https://github.com/stripe/stripe-mock) instead of the real Stripe API.

Every service uses the same `domain`, `application` and `infrastructure` package layout and exposes health checks and Prometheus metrics. Database schemas are managed with Liquibase.

## Repositories

**Platform and tooling**

| Repository | What it is |
| --- | --- |
| [glow-devops](https://github.com/glow-bas-project/glow-devops) | Shared infrastructure: the Docker Compose stack, Helm charts, Keycloak realm, PostgreSQL setup and the CI/CD templates every other repository uses |
| [glow-gradle-plugin](https://github.com/glow-bas-project/glow-gradle-plugin) | Gradle plugin that adds `glowBuild`, which rebuilds one service's image and restarts only that container in the local stack |
| [glow-bruno](https://github.com/glow-bas-project/glow-bruno) | [Bruno](https://www.usebruno.com/) API collection for every service, with local, staging and production environments and seed data |

**Frontend**

| Repository | What it is |
| --- | --- |
| [glow-ui](https://github.com/glow-bas-project/glow-ui) | React, Vite and Tailwind web app with Keycloak login, covering the customer ordering flow and the restaurant dashboard |

**Backend services**

| Repository | What it is |
| --- | --- |
| [glow-user-service](https://github.com/glow-bas-project/glow-user-service) | User accounts and roles (customer, courier, restaurant, admin); creates the user automatically on their first login |
| [glow-restaurant-service](https://github.com/glow-bas-project/glow-restaurant-service) | Restaurants, addresses and opening hours; receives orders and reports their status |
| [glow-menu-service](https://github.com/glow-bas-project/glow-menu-service) | Menu items for each restaurant, grouped by food category |
| [glow-cart-service](https://github.com/glow-bas-project/glow-cart-service) | Each customer's shopping cart |
| [glow-order-service](https://github.com/glow-bas-project/glow-order-service) | Checkout and the order lifecycle; coordinates payment and the restaurant |
| [glow-payment-service](https://github.com/glow-bas-project/glow-payment-service) | Payment intents and transfers through the Stripe SDK, against stripe-mock |
| [glow-courier-service](https://github.com/glow-bas-project/glow-courier-service) | Courier profiles and availability (in progress) |

## Tech stack

- **Backend:** Java 25, Quarkus, Hibernate ORM with Panache, Liquibase, MapStruct
- **Frontend:** React, Vite, Tailwind CSS, React Router
- **Data and auth:** PostgreSQL, Keycloak (OpenID Connect)
- **Containers and orchestration:** Docker, Docker Compose, Kubernetes, Helm
- **Routing:** Traefik, Envoy Gateway (Gateway API)
- **CI/CD:** GitLab CI with shared, versioned pipeline templates
- **Testing:** JUnit 5, Mockito, REST Assured, Playwright

## How code gets to production

Every service and the UI pull in the same versioned pipeline from `glow-devops`. On each merge to `main` it:

1. builds and tests the code,
2. publishes a Docker image for both `amd64` and `arm64`,
3. commits the new image tag to the Helm values in `glow-devops`,
4. rolls the new image out to the university's Kubernetes cluster,
5. runs Playwright smoke tests against the live deployment,
6. rolls back to the previous image automatically if those tests fail.

## Running it locally

Everything starts from [glow-devops](https://github.com/glow-bas-project/glow-devops): Docker Compose runs the whole stack on one machine, and the Helm charts run it on a local [k3d](https://k3d.io/) cluster. The container images are hosted on Aarhus University's GitLab registry, so pulling them requires an AU GitLab account.

## Team

- Sarah Rešić ([@sresix](https://github.com/sresix))
- Artūrs Siliņš ([@ASilins](https://github.com/ASilins))
- Edin Šabić ([@edinsabic](https://github.com/edinsabic))
