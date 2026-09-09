# Transforming an API Field with Kong Gateway OSS 3.5

## 1. Overview

Kong Gateway OSS 3.5 can transform an incoming API request before it is forwarded to the upstream application. The primary plugin for this purpose is the **Request Transformer** plugin.

For example, an external client may send:

```json
{
  "firstName": "John",
  "lastName": "Smith",
  "status": "A"
}
```

while the legacy backend expects:

```json
{
  "first_name": "John",
  "last_name": "Smith",
  "status": "ACTIVE"
}
```

Kong can perform simple request transformations such as:

- Rename a field
- Remove a field
- Add a field
- Replace a field value
- Add or replace headers
- Transform query-string parameters
- Use templates to construct values

The Request Transformer plugin performs transformations before the request is sent to the upstream service. Kong documents the transformation order as **remove → rename → replace → add → append**.

---

# 2. Example Architecture

```text
Client
   |
   | POST /customer
   | JSON:
   | {
   |   "firstName": "John",
   |   "lastName": "Smith",
   |   "status": "A"
   | }
   |
   v
+-------------------+
| Kong Gateway 3.5  |
|                   |
| Route             |
|       |           |
| Request           |
| Transformer       |
+---------+---------+
          |
          | Transformed JSON
          |
          v
+-------------------+
| Legacy Customer   |
| API               |
|                   |
| expects:          |
| first_name        |
| last_name         |
| status=ACTIVE     |
+-------------------+
```

---

# 3. Request Transformer Plugin

The OSS Request Transformer plugin can be attached to:

- A Service
- A Route
- A Consumer
- Globally

For most API-specific transformations, attaching the plugin to the **Route** or **Service** is preferable because the transformation is limited to the API that requires it.

---

# 4. Rename a Field

Assume the client sends:

```json
{
  "firstName": "John"
}
```

but the upstream API expects:

```json
{
  "first_name": "John"
}
```

Configure the Request Transformer plugin with:

```json
{
  "name": "request-transformer",
  "config": {
    "rename": {
      "body": [
        "firstName:first_name"
      ]
    }
  }
}
```

The syntax is:

```text
old-field:new-field
```

The Kong documentation provides `rename.body` for renaming request body parameters.

---

# 5. Add the Transformation to a Route

Assume the Kong Route is named:

```text
customer-api-route
```

Use the Admin API:

```bash
curl -i -X POST \
  http://localhost:8001/routes/customer-api-route/plugins \
  --header "Content-Type: application/json" \
  --data '{
    "name": "request-transformer",
    "config": {
      "rename": {
        "body": [
          "firstName:first_name"
        ]
      }
    }
  }'
```

The transformation now applies to requests matching that Route.

---

# 6. Transform Multiple Fields

You can rename multiple fields in the same plugin.

Example:

```json
{
  "firstName": "John",
  "lastName": "Smith",
  "postalCode": "01730"
}
```

Configure:

```json
{
  "name": "request-transformer",
  "config": {
    "rename": {
      "body": [
        "firstName:first_name",
        "lastName:last_name",
        "postalCode:postal_code"
      ]
    }
  }
}
```

The upstream receives the corresponding renamed fields.

---

# 7. Replace a Field Value

Sometimes the field name is correct, but its value must be changed.

For example, the client sends:

```json
{
  "status": "A"
}
```

while the upstream application expects:

```json
{
  "status": "ACTIVE"
}
```

Configure:

```json
{
  "name": "request-transformer",
  "config": {
    "replace": {
      "body": [
        "status:ACTIVE"
      ]
    }
  }
}
```

This replaces the existing body parameter value.

---

# 8. Add a New Field

If the client does not provide a field but the backend requires one, use `add`.

For example, add:

```json
{
  "source": "KONG"
}
```

Configure:

```json
{
  "name": "request-transformer",
  "config": {
    "add": {
      "body": [
        "source:KONG"
      ]
    }
  }
}
```

This is useful when an upstream application requires a value that should be supplied by the API gateway.

---

# 9. Remove a Field

If the client sends a field that the backend does not need:

```json
{
  "firstName": "John",
  "lastName": "Smith",
  "debug": "true"
}
```

Remove `debug`:

```json
{
  "name": "request-transformer",
  "config": {
    "remove": {
      "body": [
        "debug"
      ]
    }
  }
}
```

---

# 10. Complete Transformation Example

Suppose the client sends:

```json
{
  "firstName": "John",
  "lastName": "Smith",
  "status": "A",
  "debug": "true"
}
```

The legacy application expects:

```json
{
  "first_name": "John",
  "last_name": "Smith",
  "status": "ACTIVE",
  "source": "KONG"
}
```

A single Request Transformer configuration can perform the transformation:

```json
{
  "name": "request-transformer",
  "config": {
    "remove": {
      "body": [
        "debug"
      ]
    },
    "rename": {
      "body": [
        "firstName:first_name",
        "lastName:last_name"
      ]
    },
    "replace": {
      "body": [
        "status:ACTIVE"
      ]
    },
    "add": {
      "body": [
        "source:KONG"
      ]
    }
  }
}
```

The transformation order is important:

```text
remove
   |
   v
rename
   |
   v
replace
   |
   v
add
   |
   v
append
```

---

# 11. Configure Through the Kong Admin API

If the Route already exists:

```bash
curl -i -X POST \
  http://localhost:8001/routes/customer-api-route/plugins \
  --header "Content-Type: application/json" \
  --data '{
    "name": "request-transformer",
    "config": {
      "remove": {
        "body": [
          "debug"
        ]
      },
      "rename": {
        "body": [
          "firstName:first_name",
          "lastName:last_name"
        ]
      },
      "replace": {
        "body": [
          "status:ACTIVE"
        ]
      },
      "add": {
        "body": [
          "source:KONG"
        ]
      }
    }
  }'
```

Verify the plugin:

```bash
curl http://localhost:8001/routes/customer-api-route/plugins
```

---

# 12. Attach the Plugin to a Service Instead

If every Route associated with a Service needs the same transformation, attach the plugin to the Service.

```bash
curl -i -X POST \
  http://localhost:8001/services/customer-api/plugins \
  --header "Content-Type: application/json" \
  --data '{
    "name": "request-transformer",
    "config": {
      "rename": {
        "body": [
          "firstName:first_name"
        ]
      }
    }
  }'
```

Use a Service-level plugin when the transformation is common to all Routes for that Service.

Use a Route-level plugin when the transformation is specific to one API endpoint.

---

# 13. Transform a Header

Field transformation does not have to occur in the JSON body.

For example, add:

```text
X-Source-System: KONG
```

to the upstream request:

```bash
curl -i -X POST \
  http://localhost:8001/routes/customer-api-route/plugins \
  --header "Content-Type: application/json" \
  --data '{
    "name": "request-transformer",
    "config": {
      "add": {
        "headers": [
          "X-Source-System:KONG"
        ]
      }
    }
  }'
```

Kong will add the header before proxying the request to the upstream.

---

# 14. Transform a Query Parameter

Suppose the client calls:

```text
/customer?customerId=12345
```

but the backend expects:

```text
/customer?id=12345
```

A Request Transformer can rename the query parameter:

```json
{
  "name": "request-transformer",
  "config": {
    "rename": {
      "querystring": [
        "customerId:id"
      ]
    }
  }
}
```

The request forwarded upstream becomes conceptually:

```text
/customer?id=12345
```

---

# 15. JSON Request Body Considerations

For JSON body transformations, make sure the client sends:

```http
Content-Type: application/json
```

For example:

```bash
curl -i -X POST \
  http://localhost:8000/customer \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "John",
    "lastName": "Smith",
    "status": "A"
  }'
```

Kong receives the request, performs the configured transformation, and forwards the transformed request to the upstream.

---

# 16. Request Transformer vs Request Transformer Advanced

Kong provides both:

```text
request-transformer
request-transformer-advanced
```

The basic Request Transformer is appropriate for straightforward transformations such as:

```text
Rename
Remove
Replace
Add
Append
```

The Advanced plugin provides additional capabilities for more sophisticated JSON transformations. Kong's current documentation describes support for nested JSON objects and arrays, and body transformations require the request `Content-Type` to be `application/json`.

For example, an advanced transformation can address nested structures such as:

```json
{
  "customer": {
    "name": "John",
    "address": {
      "zip": "01730"
    }
  }
}
```

For complex JSON mappings, evaluate the Advanced plugin rather than trying to force a large transformation into the basic plugin.

---

# 17. Example: Nested JSON Transformation

Suppose the incoming request is:

```json
{
  "customer": {
    "name": "John",
    "address": {
      "zip": "01730"
    }
  }
}
```

If the upstream requires a different JSON structure, simple field renaming may not be sufficient.

A more complex transformation could require:

```text
customer.address.zip
        |
        v
customer.postalCode
```

This is a use case where the **Request Transformer Advanced** plugin is more appropriate than the basic plugin.

---

# 18. Declarative Kong Configuration

The transformation can also be defined in a declarative `kong.yml`.

Example:

```yaml
_format_version: "3.0"
_transform: true

plugins:
  - name: request-transformer
    route: customer-api-route
    config:
      remove:
        body:
          - debug

      rename:
        body:
          - firstName:first_name
          - lastName:last_name

      replace:
        body:
          - status:ACTIVE

      add:
        body:
          - source:KONG
```

This approach is useful when Kong configuration is maintained in source control and promoted between environments.

---

# 19. Testing the Transformation

Start with the original request:

```bash
curl -i -X POST \
  http://localhost:8000/customer \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "John",
    "lastName": "Smith",
    "status": "A",
    "debug": "true"
  }'
```

The backend should receive approximately:

```json
{
  "first_name": "John",
  "last_name": "Smith",
  "status": "ACTIVE",
  "source": "KONG"
}
```

Use an HTTP echo service or inspect the backend application's request logs to verify the transformation.

---

# 20. Troubleshooting

## Transformation Does Not Occur

Check whether the plugin is installed/enabled:

```bash
curl http://localhost:8001/plugins
```

Check the Route:

```bash
curl http://localhost:8001/routes/customer-api-route
```

Check the Route plugins:

```bash
curl http://localhost:8001/routes/customer-api-route/plugins
```

Verify:

- The request matches the intended Route.
- The plugin is attached to the correct Route or Service.
- The request uses the expected HTTP method.
- The body is sent with `Content-Type: application/json`.
- The field name exactly matches the configured field.
- The upstream API is receiving the request.

---

# 21. Common Mistake: `localhost` in Docker

If Kong is running in Docker and the upstream API is another Docker container, avoid:

```text
http://localhost:8080
```

unless the API actually runs inside the Kong container.

Instead, use the Docker service/container DNS name:

```text
http://customer-api:8080
```

Example architecture:

```text
kong-net
   |
   +-- kong
   |
   +-- customer-api
```

The Kong Service should use:

```text
http://customer-api:8080
```

---

# 22. Recommended Approach for an Air-Gapped/Government Environment

For an isolated or government API environment, use Kong transformation primarily as an **API mediation capability**, not as the primary location for complex business logic.

A recommended pattern is:

```text
External API
     |
     v
Kong Gateway
     |
     +-- Authentication
     |
     +-- Authorization
     |
     +-- Rate Limiting
     |
     +-- Request Transformation
     |
     +-- Logging
     |
     v
Internal API
```

Use Kong for relatively simple compatibility transformations such as:

```text
REST field rename
REST field addition
REST field removal
Header transformation
Query parameter transformation
Legacy API compatibility
```

Keep complex business rules, data validation, calculations, and major schema conversions in the application or a dedicated integration/transformation service.

---

# 23. SOAP/XML Consideration

The Request Transformer plugin is primarily suited to HTTP request transformations and JSON/form-style parameters. It should not be treated as a general-purpose SOAP-to-REST XML transformation engine.

For a SOAP-to-REST modernization architecture, a better pattern is:

```text
REST Client
    |
    v
Kong Gateway
    |
    | REST
    v
Transformation / Mediation Layer
    |
    | SOAP/XML
    v
Legacy SOAP Service
```

Possible mediation technologies include:

- DataPower
- Custom Java/Spring service
- .NET API
- Integration platform
- Dedicated transformation service

Kong can remain the API gateway responsible for exposure, security, routing, rate limiting, and gateway-level transformations.

---

# 24. Recommended Field Transformation Pattern

For an API modernization effort, use a transformation table to document mappings.

| Client Field | Kong Transformation | Upstream Field | Transformation Type |
|---|---|---|---|
| `firstName` | Rename | `first_name` | Rename |
| `lastName` | Rename | `last_name` | Rename |
| `status` | Replace `A` → `ACTIVE` | `status` | Value replacement |
| `debug` | Remove | — | Remove |
| — | Add `source=KONG` | `source` | Add |
| `customerId` | Rename | `id` | Query parameter rename |

This makes the API contract and gateway configuration traceable.

---

# 25. Summary

Kong Gateway OSS 3.5 can perform simple API field transformations using the **Request Transformer** plugin. For a field-level change, the most common operations are `rename.body`, `replace.body`, `add.body`, and `remove.body`. The plugin can be attached to a specific Route or Service through the Kong Admin API or represented in declarative configuration.

For simple API compatibility mappings, Kong is a good location for the transformation because it keeps legacy-system details behind the gateway. For complex nested JSON transformations, extensive business logic, or SOAP/XML conversion, use the appropriate advanced transformation or a dedicated mediation/application layer rather than making Kong responsible for business logic.
