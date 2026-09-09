# Adding an API to Kong Gateway OSS 3.5

## 1. Overview

Kong Gateway OSS 3.5 exposes backend applications through two primary
configuration objects:

-   **Service** -- represents the upstream API or application.
-   **Route** -- defines how clients reach the Service through Kong.

The basic request flow is:

``` text
API Client
    |
    |  HTTP/HTTPS request
    v
Kong Proxy
    |
    |  Route matches request
    v
Kong Service
    |
    |  Forwards request
    v
Upstream API
```

For a typical API, you will create:

1.  An upstream API or application.
2.  A Kong Service pointing to the upstream API.
3.  A Kong Route exposing the Service.
4.  Optional plugins for authentication, authorization, rate limiting,
    logging, transformation, or other policies.
5.  Tests through the Kong proxy port.

Kong's Admin API normally listens on port **8001**, while the HTTP proxy
normally listens on **8000**. HTTPS proxy traffic normally uses
**8443**.

------------------------------------------------------------------------

## 2. Prerequisites

Before adding an API, verify that Kong is running.

### Check Kong

``` bash
curl http://localhost:8001/
```

A successful response returns Kong configuration and version
information.

You can also check the Kong status endpoint:

``` bash
curl http://localhost:8001/status
```

### Check the Kong proxy

``` bash
curl http://localhost:8000/
```

The proxy request may return `404` if no Route matches. That does not
necessarily mean Kong is down; it can simply mean that no API has been
configured for the requested path.

### Important ports

  Port   Purpose
  ------ ------------------------------
  8000   HTTP proxy
  8443   HTTPS proxy
  8001   Admin API HTTP
  8444   Admin API HTTPS
  8002   Kong Manager OSS, if enabled

**Security note:** Do not expose the Kong Admin API (8001/8444) to
untrusted networks. The Admin API provides administrative control over
Kong.

------------------------------------------------------------------------

# 3. Example API

For this guide, assume an existing REST API is running at:

``` text
http://api-server:8080/customer
```

We want Kong clients to access it through:

``` text
http://localhost:8000/customer
```

The configuration will be:

``` text
Client
   |
   | GET /customer
   v
Kong :8000
   |
   | Route: /customer
   v
Service: customer-api
   |
   | http://api-server:8080/customer
   v
Backend API
```

------------------------------------------------------------------------

# 4. Add an API Using the Kong Admin API

The most direct way to configure Kong OSS 3.5 is through the Admin API.

There are two primary operations:

1.  Create the **Service**.
2.  Create the **Route**.

------------------------------------------------------------------------

## 4.1 Create the Service

A Service identifies the backend application.

Using `curl`:

``` bash
curl -i -X POST http://localhost:8001/services \
  --data name=customer-api \
  --data url=http://api-server:8080/customer
```

A successful request should return:

``` text
HTTP/1.1 201 Created
```

The Service can now be viewed with:

``` bash
curl http://localhost:8001/services/customer-api
```

Example response:

``` json
{
  "name": "customer-api",
  "host": "api-server",
  "port": 8080,
  "protocol": "http",
  "path": "/customer",
  "enabled": true
}
```

The `url` field is a convenient way to specify the Service's protocol,
host, port, and path.

------------------------------------------------------------------------

# 5. Create a Route

A Route determines how clients access the Service.

Create a Route named `customer-api-route`:

``` bash
curl -i -X POST http://localhost:8001/services/customer-api/routes \
  --data name=customer-api-route \
  --data paths[]=/customer
```

A successful response should return:

``` text
HTTP/1.1 201 Created
```

You can verify the Route:

``` bash
curl http://localhost:8001/services/customer-api/routes
```

You can also list all Routes:

``` bash
curl http://localhost:8001/routes
```

------------------------------------------------------------------------

# 6. Test the API Through Kong

Once the Service and Route are configured, send a request to the Kong
proxy.

``` bash
curl -i http://localhost:8000/customer
```

The request flow is:

``` text
GET http://localhost:8000/customer
        |
        v
Kong Route /customer
        |
        v
Service customer-api
        |
        v
http://api-server:8080/customer
```

If the upstream API returns:

``` json
{
  "status": "success"
}
```

the same response should normally be returned through Kong.

------------------------------------------------------------------------

# 7. Create the Service Using JSON

JSON can be easier to use for repeatable configuration.

``` bash
curl -i -X POST http://localhost:8001/services \
  -H "Content-Type: application/json" \
  -d '{
    "name": "customer-api",
    "url": "http://api-server:8080/customer"
  }'
```

Create the Route:

``` bash
curl -i -X POST http://localhost:8001/services/customer-api/routes \
  -H "Content-Type: application/json" \
  -d '{
    "name": "customer-api-route",
    "paths": ["/customer"],
    "methods": ["GET", "POST"]
  }'
```

The `methods` property is optional. If omitted, the Route can match the
supported HTTP methods unless other Route restrictions apply.

------------------------------------------------------------------------

# 8. Configure Route Options

A Route can be made more specific.

For example:

``` json
{
  "name": "customer-api-route",
  "paths": ["/customer"],
  "methods": ["GET"],
  "protocols": ["http", "https"],
  "strip_path": false
}
```

Important Route attributes include:

  -----------------------------------------------------------------------
  Attribute                           Purpose
  ----------------------------------- -----------------------------------
  `name`                              Unique name for the Route

  `paths`                             URL paths matched by the Route

  `methods`                           HTTP methods accepted

  `hosts`                             Hostnames matched by the Route

  `protocols`                         HTTP and/or HTTPS

  `strip_path`                        Determines whether the matched
                                      Route path is removed before
                                      forwarding

  `tags`                              Metadata used for organization
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 9. Understanding `strip_path`

`strip_path` is particularly important when exposing existing APIs.

Assume the Service is:

``` text
http://api-server:8080/customer
```

and the Route is:

``` text
/customer
```

With:

``` json
"strip_path": true
```

Kong can remove the matching Route prefix before forwarding the request.

With:

``` json
"strip_path": false
```

Kong preserves the Route path when forwarding.

Always test the resulting upstream URL because the correct setting
depends on how the backend API is implemented.

------------------------------------------------------------------------

# 10. Use a Host-Based Route

Instead of routing based only on a path, you can route based on a
hostname.

For example:

``` bash
curl -i -X POST http://localhost:8001/services/customer-api/routes \
  -H "Content-Type: application/json" \
  -d '{
    "name": "customer-api-host-route",
    "hosts": ["customer-api.example.mil"],
    "paths": ["/customer"],
    "protocols": ["http", "https"]
  }'
```

The client would then use:

``` text
https://customer-api.example.mil/customer
```

DNS or an appropriate hosts-file entry must resolve the hostname to
Kong.

------------------------------------------------------------------------

# 11. Add Authentication

Kong plugins can be attached at the global, Service, or Route level.

For example, to use API key authentication, enable the Key
Authentication plugin on the Service:

``` bash
curl -i -X POST http://localhost:8001/services/customer-api/plugins \
  --data name=key-auth
```

The plugin can then be combined with a Kong Consumer and credential.

Create a Consumer:

``` bash
curl -i -X POST http://localhost:8001/consumers \
  --data username=customer-client
```

Create an API key:

``` bash
curl -i -X POST http://localhost:8001/consumers/customer-client/key-auth \
  --data key=my-secret-key
```

The client can then call:

``` bash
curl -i http://localhost:8000/customer \
  -H "apikey: my-secret-key"
```

For production systems, use an appropriate enterprise identity
architecture rather than embedding long-lived credentials in scripts.

------------------------------------------------------------------------

# 12. Add Rate Limiting

A rate-limiting plugin can be applied to a Service or Route.

Example:

``` bash
curl -i -X POST http://localhost:8001/services/customer-api/plugins \
  --data name=rate-limiting \
  --data config.minute=100 \
  --data config.policy=local
```

This example limits requests to 100 requests per minute using Kong's
local rate-limiting policy.

For multiple Kong nodes, select the rate-limiting policy carefully
because local counters are maintained independently by each node.

------------------------------------------------------------------------

# 13. Add Request/Response Logging

Logging plugins can be used to provide visibility into API traffic.

For example:

``` bash
curl -i -X POST http://localhost:8001/services/customer-api/plugins \
  --data name=file-log \
  --data config.path=/tmp/customer-api.log
```

In an operational environment, consider centralized logging instead of
writing logs only to the Kong container or host.

Common enterprise logging destinations include:

-   Syslog
-   HTTP logging endpoints
-   Kafka
-   SIEM platforms
-   ELK/OpenSearch
-   Security monitoring platforms

------------------------------------------------------------------------

# 14. Add an Upstream for Multiple API Servers

For high availability or load balancing, create an Upstream instead of
pointing the Service directly to one server.

Example:

``` bash
curl -i -X POST http://localhost:8001/upstreams \
  --data name=customer-api-upstream
```

Add two Targets:

``` bash
curl -i -X POST http://localhost:8001/upstreams/customer-api-upstream/targets \
  --data target=api-server-1:8080 \
  --data weight=100
```

``` bash
curl -i -X POST http://localhost:8001/upstreams/customer-api-upstream/targets \
  --data target=api-server-2:8080 \
  --data weight=100
```

Create the Service pointing to the Upstream:

``` bash
curl -i -X POST http://localhost:8001/services \
  --data name=customer-api \
  --data host=customer-api-upstream \
  --data port=8080 \
  --data protocol=http
```

The resulting architecture becomes:

``` text
                  +----------------+
                  |  Kong Gateway  |
                  +-------+--------+
                          |
                    Customer API
                       Service
                          |
                  Customer Upstream
                    /           \
                   /             \
                  v               v
          API Server 1       API Server 2
```

This approach is useful when the API has multiple instances and requires
load balancing or health checking.

------------------------------------------------------------------------

# 15. Kong Manager OSS

If Kong Manager OSS is enabled, the graphical interface can be used to
configure Services and Routes.

A typical local installation exposes Kong Manager on:

``` text
http://localhost:8002
```

The basic workflow is:

1.  Open Kong Manager.
2.  Navigate to **Services**.
3.  Select **Add New Service**.
4.  Enter the Service name.
5.  Enter the upstream URL.
6.  Save the Service.
7.  Open the Service.
8.  Select **Routes**.
9.  Select **Add Route**.
10. Enter the Route path.
11. Select the HTTP methods if required.
12. Save the Route.
13. Test the API through the Kong proxy.

For an OSS 3.5 deployment, the exact Manager availability and UI options
depend on how Kong Manager was installed and configured.

------------------------------------------------------------------------

# 16. Verify the Configuration

List Services:

``` bash
curl http://localhost:8001/services
```

List Routes:

``` bash
curl http://localhost:8001/routes
```

List Plugins:

``` bash
curl http://localhost:8001/plugins
```

Inspect the Service:

``` bash
curl http://localhost:8001/services/customer-api
```

Inspect the Route:

``` bash
curl http://localhost:8001/services/customer-api/routes
```

Test the proxy:

``` bash
curl -i http://localhost:8000/customer
```

------------------------------------------------------------------------

# 17. Troubleshooting

## 17.1 404 Not Found

If:

``` bash
curl http://localhost:8000/customer
```

returns:

``` text
404 Not Found
```

check:

``` bash
curl http://localhost:8001/routes
```

Verify that the Route contains:

``` text
/customer
```

Also verify that the Route is associated with the correct Service.

------------------------------------------------------------------------

## 17.2 502 Bad Gateway

A `502 Bad Gateway` generally indicates that Kong could not successfully
communicate with the upstream.

Check the Service:

``` bash
curl http://localhost:8001/services/customer-api
```

Verify:

-   Hostname
-   IP address
-   Port
-   Protocol
-   Backend application status
-   Docker network connectivity
-   Firewall rules
-   DNS resolution

If Kong is running inside Docker, remember that:

``` text
localhost
```

inside the Kong container means the Kong container itself, not the host
computer.

For Docker deployments, the Service should normally use the backend
container's Docker DNS name, for example:

``` text
http://my-api:8080
```

rather than:

``` text
http://localhost:8080
```

------------------------------------------------------------------------

## 17.3 Connection Refused

If Kong cannot connect to the upstream:

``` text
connection refused
```

verify that the backend is listening on the expected interface and port.

For Docker:

``` bash
docker ps
```

Check the network:

``` bash
docker network inspect kong-net
```

Test connectivity from the Kong container when appropriate:

``` bash
docker exec -it <kong-container> sh
```

Then test the backend:

``` bash
curl http://my-api:8080/health
```

------------------------------------------------------------------------

## 17.4 Admin API Connection Failure

If this fails:

``` bash
curl http://localhost:8001/
```

verify the Kong container:

``` bash
docker ps
```

Check logs:

``` bash
docker logs <kong-container>
```

Also verify that port 8001 is mapped from the container to the host.

------------------------------------------------------------------------

# 18. Docker Example

For a Docker-based Kong deployment, the basic relationship should look
like:

``` text
Docker Host
|
+-- kong-net
    |
    +-- kong
    |    +-- 8000  Proxy
    |    +-- 8001  Admin API
    |    +-- 8002  Kong Manager
    |
    +-- customer-api
         +-- 8080  REST API
```

The Kong Service should then use:

``` text
http://customer-api:8080
```

rather than:

``` text
http://localhost:8080
```

Example:

``` bash
curl -i -X POST http://localhost:8001/services \
  --data name=customer-api \
  --data url=http://customer-api:8080
```

Then:

``` bash
curl -i -X POST http://localhost:8001/services/customer-api/routes \
  --data name=customer-api-route \
  --data paths[]=/customer
```

Test:

``` bash
curl http://localhost:8000/customer
```

------------------------------------------------------------------------

# 19. Recommended API Onboarding Process

For an operational API management environment, use the following
sequence:

``` text
1. Identify API
       |
       v
2. Identify upstream endpoint
       |
       v
3. Define API owner
       |
       v
4. Create Kong Service
       |
       v
5. Create Kong Route
       |
       v
6. Configure authentication
       |
       v
7. Configure authorization
       |
       v
8. Configure rate limiting
       |
       v
9. Configure logging/monitoring
       |
       v
10. Test API
       |
       v
11. Document API
       |
       v
12. Promote through environments
```

Recommended metadata for each API includes:

  Field            Example
  ---------------- -------------------------------------
  API Name         Customer API
  API Version      v1
  Service Name     customer-api
  Route            `/api/v1/customer`
  Protocol         HTTPS
  Owner            Application Team
  Data Owner       Program Office
  Classification   Appropriate security classification
  Authentication   Enterprise IAM
  Authorization    RBAC/ABAC
  Rate Limit       1000 requests/minute
  Logging          Central SIEM
  Backend          Customer application
  Environment      DEV/TEST/PROD
  Documentation    OpenAPI specification

------------------------------------------------------------------------

# 20. Recommended Naming Convention

Use consistent names across Kong.

Example:

``` text
Service:
customer-api-v1

Route:
customer-api-v1-route

Upstream:
customer-api-v1-upstream

Consumer:
customer-api-client

Plugin:
customer-api-v1-auth
```

For larger environments, include the program, domain, and environment:

``` text
<program>-<domain>-<api>-<version>-<environment>
```

Example:

``` text
alcm-logistics-maintenance-v1-prod
```

------------------------------------------------------------------------

# 21. REST API Versioning

A recommended Route structure is:

``` text
/api/v1/customer
/api/v2/customer
```

This allows different API versions to coexist.

For example:

``` text
/api/v1/customer
        |
        v
customer-api-v1
        |
        v
Customer Application V1
```

and:

``` text
/api/v2/customer
        |
        v
customer-api-v2
        |
        v
Customer Application V2
```

Kong can therefore provide a stable API entry point while backend
implementations evolve independently.

------------------------------------------------------------------------

# 22. Declarative Configuration Alternative

For repeatable deployments, Kong can also be configured using a
declarative YAML file.

Example `kong.yml`:

``` yaml
_format_version: "3.0"
_transform: true

services:
  - name: customer-api
    url: http://customer-api:8080

    routes:
      - name: customer-api-route
        paths:
          - /customer
        methods:
          - GET
          - POST
```

This configuration can be stored in source control and promoted through
environments.

For DB-less Kong, the declarative configuration becomes the primary
configuration mechanism. For database-backed Kong, the Admin API can be
used to configure entities dynamically.

------------------------------------------------------------------------

# 23. Complete Quick-Start Script

The following commands create a basic API in Kong:

``` bash
# 1. Create Service
curl -i -X POST http://localhost:8001/services \
  --data name=customer-api \
  --data url=http://customer-api:8080

# 2. Create Route
curl -i -X POST http://localhost:8001/services/customer-api/routes \
  --data name=customer-api-route \
  --data paths[]=/customer

# 3. Verify Service
curl http://localhost:8001/services/customer-api

# 4. Verify Route
curl http://localhost:8001/services/customer-api/routes

# 5. Test through Kong
curl -i http://localhost:8000/customer
```

------------------------------------------------------------------------

# 24. API Onboarding Checklist

## Kong Configuration

-   [ ] Kong Gateway 3.5 is running.
-   [ ] Admin API is accessible from the administration network.
-   [ ] Proxy port is accessible to API clients.
-   [ ] Service has been created.
-   [ ] Service points to the correct upstream.
-   [ ] Route has been created.
-   [ ] Route points to the correct Service.
-   [ ] HTTP methods are restricted as required.
-   [ ] HTTPS is configured for production traffic.
-   [ ] Authentication is configured.
-   [ ] Authorization is configured.
-   [ ] Rate limiting is configured.
-   [ ] Logging is configured.
-   [ ] Monitoring is configured.
-   [ ] API documentation is available.
-   [ ] API ownership has been established.
-   [ ] Security classification/handling requirements have been
    identified.
-   [ ] API has been tested through Kong.

## Testing

``` bash
# Kong health
curl http://localhost:8001/status

# Services
curl http://localhost:8001/services

# Routes
curl http://localhost:8001/routes

# Proxy test
curl -i http://localhost:8000/customer
```

------------------------------------------------------------------------

# 25. Summary

Adding an API to Kong Gateway OSS 3.5 primarily consists of creating a
**Service** and a **Route**. The Service represents the backend API,
while the Route defines how clients access that Service through Kong.
Additional Kong plugins can then provide authentication, authorization,
rate limiting, logging, transformations, and other API-management
capabilities.

For a development or proof-of-concept environment, the Kong Admin API is
the simplest approach. For a controlled enterprise or government
environment, use repeatable configuration, source control, automated
deployment, strong protection of the Admin API, centralized logging and
monitoring, enterprise identity integration, and documented API
governance. Kong's declarative configuration is particularly useful when
API configurations need to be promoted consistently across DEV, TEST,
and PROD environments.
