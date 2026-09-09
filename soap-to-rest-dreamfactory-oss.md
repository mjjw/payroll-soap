# Converting a SOAP API to REST in DreamFactory OSS

DreamFactory can auto-generate a fully documented REST API on top of an existing SOAP web service. You point it at a WSDL, and it exposes JSON-based REST endpoints that internally translate to and from SOAP/XML — no manual endpoint coding required.

## Prerequisites

- A running DreamFactory OSS instance (Docker, source install, or hosted trial) with admin access.
- The WSDL URL (or a local WSDL file) for the SOAP service you want to wrap.
- Any credentials the SOAP service requires (basic auth, WS-Security tokens, etc.), if applicable.

## Step 1: Open the API Generation screen

1. Log in to the DreamFactory admin console.
2. Go to **API Generation & Connections** in the left navigation.
3. Click the **Network** dropdown under **API Types**, then click the **+** button to create a new service.

## Step 2: Create a SOAP service

1. From the service type list, select **SOAP**.
2. Fill in:
   - **Name** — a short identifier; this becomes part of the generated API's URL path, so keep it URL-friendly (e.g. `weather_soap`).
   - **Label** — a human-friendly display name (admin UI only).
   - **Description** — optional notes for other admins (admin UI only).

## Step 3: Point it at your WSDL

1. Switch to the **Config** tab for the new service.
2. Enter the **WSDL URL** (or upload a local WSDL file if the service isn't publicly reachable).
3. Add any required authentication details for the SOAP endpoint (username/password, headers, WS-Security settings) in the config options provided.
4. Click **Save**.

DreamFactory parses the WSDL and automatically builds REST endpoints that map to each SOAP operation.

## Step 4: Review the auto-generated REST API

1. Open **API Docs** for your new service — DreamFactory generates a live Swagger/OpenAPI definition.
2. Each SOAP operation now appears as a REST endpoint. Expand one to see the expected JSON request body and JSON response shape.
3. Use the **Try it out** feature in API Docs to fire a test request directly from the browser.

Behind the scenes, DreamFactory converts your JSON request into the SOAP/XML envelope the backend service expects, sends it, and converts the SOAP/XML response back into JSON for you.

## Step 5: Lock down access with roles

1. Go to **Roles & Access** and create (or edit) a role.
2. Grant that role access to specific verbs/endpoints on your new SOAP-backed service — you don't have to expose every operation.
3. Generate an **API key** tied to an app, and/or issue **JWTs** for user-based access, so consuming apps only see what they're permitted to.

## Step 6 (optional): Transform data with scripting

If the SOAP response shape doesn't match what your client app wants, or you need custom logic (renaming fields, filtering, combining calls), use DreamFactory's scripting hooks:

- Supported languages: **Node.js, PHP, Python** (availability can depend on installed script engines).
- Scripts can run pre-process (before the request hits SOAP) or post-process (before the JSON response goes back to the client).
- Attach scripts under the service's **Scripts** tab at the event (endpoint) level.

## Step 7: Call your new REST API

Your client applications now call plain REST/JSON endpoints, e.g.:

```
GET https://your-dreamfactory-instance/api/v2/weather_soap/{operation}
```

with a JSON body/query params instead of hand-built XML/SOAP envelopes — while DreamFactory handles the WSDL, XML, and legacy protocol details for you.

## Notes

- Because this is OSS, double-check that the SOAP connector and scripting engines you need are enabled/installed in your build — some advanced auth schemes (e.g. WS-Security) may have more complete support in DreamFactory's commercial edition.
- Test against a non-production SOAP endpoint first, especially if the service is stateful or has side effects, since the generated REST wrapper will forward calls 1:1 unless you add scripting logic.
