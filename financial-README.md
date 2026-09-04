# Financial Service

A small demo/training REST API built with Spring Boot. It's one of four sibling services
(`personnel`, `contracts`, `financial`, `tracker`) designed to sit behind an API gateway
(the code references [Kong](https://konghq.com/products/kong-gateway)) and exercise gateway
behaviors — slow responses, randomized HTTP status codes, request echoing, IP whitelisting —
the kind of thing you'd want to test rate limiting, retries, circuit breakers, or client
resiliency against. Unlike its siblings, this one is **reactive** (WebFlux) and its
distinguishing feature is orchestrating chained/fan-out calls to the other three services.

## Stack

- Java 21, Spring Boot 3.2 (Spring WebFlux — `spring-boot-starter-webflux`, reactive/non-blocking)
- Gradle (wrapper included, no local Gradle install needed)
- [springdoc-openapi](https://springdoc.org/) WebFlux flavor for Swagger UI / OpenAPI docs
- Spring Boot Actuator (health probes enabled)
- Micrometer tracing (Brave/Zipkin bridge) — currently disabled
- Reactor Netty `WebClient` for outbound calls
- Lombok

## Running it

### Locally with Gradle

```bash
./gradlew bootRun
```

The service listens on **port 9004** (see `src/main/resources/application.yml`).

### As a container

The `Dockerfile` expects a pre-built jar rather than building it inside the image, so build
first, then containerize:

```bash
./gradlew build
docker build -t financial-service .
docker run -p 9004:9004 financial-service
```

### Once running

- Swagger UI: http://localhost:9004/swagger-ui.html
- OpenAPI spec (JSON): http://localhost:9004/v3/api-docs
- Health: http://localhost:9004/actuator/health

## Endpoints

All under `/api/v1` unless noted.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/fast` | Responds immediately with a simple message (reactive `Mono`) |
| GET | `/slow` | Sleeps 1–3s before responding |
| GET | `/who-am-i` | Returns the host's local IP address |
| GET | `/fast-random` | Responds immediately with a randomized HTTP status (200/400/403/415/500) |
| GET | `/slow-random` | Sleeps 1–3s, then responds with a randomized HTTP status |
| GET | `/chain/async` | Fires 5 concurrent calls to the other services and streams results back as Server-Sent Events as each resolves |
| GET | `/chain/sync` | Fires 5 calls to the other services sequentially, waits for each, and returns the collected list |
| GET | `/chain/random-status` | Like `/chain/sync`, but hits each service's `*-random` endpoint, so some calls will return non-200 statuses |

## How the chaining works

`ServiceAddress` defines routes reached through a Kong gateway at `http://kong:8000`:
`ALPHA`, `BETA`, `GAMMA` (the other three services, fronted by the gateway) and `OMEGA`
(this service itself). `FinancialChainRestController` picks one of the four at random for
each call and hits either its `/api/v1/fast(-random)` or `/api/v1/slow(-random)` endpoint.

**This means the chain endpoints only work when run alongside a Kong gateway** (or
something answering at that hostname/port) that proxies to `personnel`, `contracts`,
`tracker`, and this service. Commented-out alternatives in `ServiceAddress.java` show how
to point directly at `localhost:9001-9004` or at service hostnames (`alpha`, `beta`,
`gamma`, `financial`) for a docker-compose setup without a gateway in front.

## Configuration notes

- **IP whitelist filter is present but inactive.** `WhitelistIPFilter` (the WebFlux
  `WebFilter` variant) reads an `${security.whitelist-ip}` property list and, when active,
  rejects any request from an address not on the list. Its `@Component` annotation is
  commented out, so it is **not** registered as a filter right now — and no property file
  supplying `security.whitelist-ip` exists in this service (unlike `personnel`/`contracts`),
  so it would need one added before being re-enabled.
- `WebClientConfig` builds the shared `WebClient` bean with a 10s connect/read/write timeout
  via a Reactor Netty `HttpClient`.
- A commented-out SSL block in `application.yml` shows a PKCS12 keystore config, unused.
