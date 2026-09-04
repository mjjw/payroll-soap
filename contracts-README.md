# Contracts Service

A small demo/training REST API built with Spring Boot. It's one of four sibling services
(`personnel`, `contracts`, `financial`, `tracker`) designed to sit behind an API gateway
(the code references [Kong](https://konghq.com/products/kong-gateway)) and exercise gateway
behaviors — slow responses, randomized HTTP status codes, request echoing, IP whitelisting —
the kind of thing you'd want to test rate limiting, retries, circuit breakers, or client
resiliency against. This service's distinguishing feature is file hashing.

## Stack

- Java 21, Spring Boot 3.2 (Spring MVC — `spring-boot-starter-web`)
- Gradle (wrapper included, no local Gradle install needed)
- [springdoc-openapi](https://springdoc.org/) for Swagger UI / OpenAPI docs
- Spring Boot Actuator (health probes enabled)
- Micrometer tracing (Brave/Zipkin bridge) — currently disabled
- Lombok
- Apache Commons Lang3 + Commons Codec

## Running it

### Locally with Gradle

```bash
./gradlew bootRun
```

The service listens on **port 9002** (see `src/main/resources/application.yml`).

### As a container

The `Dockerfile` expects a pre-built jar rather than building it inside the image, so build
first, then containerize:

```bash
./gradlew build
docker build -t contracts-service .
docker run -p 9002:9002 contracts-service
```

### Once running

- Swagger UI: http://localhost:9002/swagger-ui.html
- OpenAPI spec (JSON): http://localhost:9002/v3/api-docs
- Health: http://localhost:9002/actuator/health

## Endpoints

All under `/api/v1` unless noted.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/fast` | Responds immediately with a simple message |
| GET | `/fast-random` | Responds immediately with a randomized HTTP status (200/400/403/415/500) |
| GET | `/slow` | Sleeps 1–3s before responding |
| GET | `/slow-random` | Sleeps 1–3s, then responds with a randomized HTTP status |
| GET | `/who-am-i` | Returns the host's local IP address |
| PUT | `/echo-body` | Echoes the request body back as plain text |
| POST | `/md5` | Multipart file upload (`f`); returns the file's MD5 hash as plain text |

## Configuration notes

- **IP whitelist filter is present but inactive.** `WhitelistIPFilter` reads an
  `${security.whitelist-ip}` property list and, when active, rejects any request from an
  address not on the list. Its `@Component` annotation is commented out, so it is **not**
  registered as a filter right now.
- The whitelist value itself lives in `src/main/resources/contracts.yml` — but note this
  file isn't referenced anywhere (no `spring.config.import`), so Spring Boot won't load it
  automatically. Even with the filter re-enabled, you'd need to import this file explicitly
  or fold its contents into `application.yml`.
- `HmacUtil` (HMAC-SHA256, hex or Base64 output) exists in the codebase but isn't wired up
  to any controller yet — it's a utility ready to be exposed by a future endpoint.
- `ErrorResponse` (message + timestamp) is defined but not currently used by any controller
  or exception handler.
- Jackson is configured for snake_case property naming, `non-null` inclusion, and a fixed
  ISO-ish date format — applies to any JSON this service returns.
- Multipart uploads are capped at 10MB per file / 15MB per request (`spring.servlet.multipart`).
- `RestConfig` exposes a `RestTemplate` bean (10s connect/read timeout) for use elsewhere in
  the app, though nothing currently calls out to other services from this one.
- `contractsBuild.zip` in the repo root looks like a stale build-output archive (compiled
  `.class` files) — safe to delete; it's not referenced by the Gradle build.
