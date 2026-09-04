# Tracker Service

A small demo/training REST API built with Spring Boot. It's one of four sibling services
(`personnel`, `contracts`, `financial`, `tracker`) designed to sit behind an API gateway
(the code references [Kong](https://konghq.com/products/kong-gateway)) and exercise gateway
behaviors — slow responses, randomized HTTP status codes, request echoing, IP whitelisting —
the kind of thing you'd want to test rate limiting, retries, circuit breakers, or client
resiliency against. This service's distinguishing feature is QR code generation.

## Stack

- Java 21, Spring Boot 3.2 (Spring MVC — `spring-boot-starter-web`)
- Gradle (wrapper included, no local Gradle install needed)
- [springdoc-openapi](https://springdoc.org/) for Swagger UI / OpenAPI docs
- Spring Boot Actuator (health probes enabled)
- Micrometer tracing (Brave/Zipkin bridge) — currently disabled
- [ZXing](https://github.com/zxing/zxing) (`com.google.zxing:javase`) for QR code generation
- Spring Cloud BOM imported (`2023.0.0`) for dependency management
- Lombok, Apache Commons Lang3

## Running it

### Locally with Gradle

```bash
./gradlew bootRun
```

The service listens on **port 9003** (see `src/main/resources/application.yml`).

### As a container

The `Dockerfile` expects a pre-built jar rather than building it inside the image, so build
first, then containerize:

```bash
./gradlew build
docker build -t tracker-service .
docker run -p 9003:9003 tracker-service
```

### Once running

- Swagger UI: http://localhost:9003/swagger-ui.html
- OpenAPI spec (JSON): http://localhost:9003/v3/api-docs
- Health: http://localhost:9003/actuator/health

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/fast` | Responds immediately with a simple message |
| GET | `/api/v1/fast-random` | Responds immediately with a randomized HTTP status (200/400/403/415/500) |
| GET | `/api/v1/slow` | Sleeps 1–3s before responding |
| GET | `/api/v1/slow-random` | Sleeps 1–3s, then responds with a randomized HTTP status |
| GET | `/api/v1/who-am-i` | Returns the host's local IP address |
| GET | `/v1/create-qr-code` | Generates a QR code image. Params: `data` (required, text to encode), `size` (16–1024, default 250), `format` (`PNG`/`JPG`/`JPEG`, default `PNG`) |

Note the QR endpoint lives at `/v1/...`, not `/api/v1/...` like the others — there's no
`@RequestMapping` prefix on `QrCodeRestController`, unlike the sibling controller.

## Configuration notes

- **IP whitelist filter is present but inactive.** `WhitelistIPFilter` reads an
  `${security.whitelist-ip}` property list and, when active, rejects any request from an
  address not on the list. Its `@Component` annotation is commented out, so it is **not**
  registered as a filter right now — and no property file supplying `security.whitelist-ip`
  exists in this service, so it would need one added before being re-enabled.
- **The H2 datasource config in `application.yml` is currently inert.** It declares
  `spring.datasource.url: jdbc:h2:mem:tracker-db` and enables the H2 console, but
  `build.gradle` doesn't include the H2 driver or any `spring-boot-starter-data-jpa`/`-jdbc`
  dependency, and no code in the project uses a `DataSource`, JPA repository, or JDBC
  template. Spring Boot won't auto-configure a datasource without a driver on the classpath,
  so these properties currently have no effect. Add `com.h2database:h2` (and a data/JDBC
  starter) if you want to actually wire up persistence here.
- `RestConfig` exposes a `RestTemplate` bean (10s connect/read timeout) for use elsewhere in
  the app, though nothing currently calls out to other services from this one.
- QR code images use a fixed dark/light color pair (`MatrixToImageConfig`) and are served
  with a 24-hour `Cache-Control` header.
