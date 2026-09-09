# Generating a REST API from a SQLite Database in DreamFactory OSS

DreamFactory can connect directly to a SQLite database file, introspect its schema, and auto-generate a fully documented REST API with CRUD (Create, Read, Update, Delete) endpoints — no backend code required. SQLite is one of the database connectors included in the open-source edition.

## Prerequisites

- A running DreamFactory OSS instance with admin access.
- A SQLite database file (`.db`, `.sqlite`, or `.sqlite3`) accessible from the DreamFactory server/container's filesystem.
- Basic familiarity with your database's table/column structure (helpful for testing, not required).

## Step 1: Make the SQLite file available to DreamFactory

If DreamFactory is running in Docker or on a separate host from your file, copy or mount the `.db` file somewhere the DreamFactory process can read (and write, if you need updates), such as a shared storage volume. Note the full path — you'll need it during setup.

## Step 2: Open the API Generation screen

1. Log in to the DreamFactory admin console.
2. Go to **API Generation & Connections** in the left navigation.
3. Click the **Database** dropdown under **API Types**.
4. Click the **+** button to create a new service.

## Step 3: Create a SQLite service

1. Search for or select the **SQLite** service type.
2. Fill in:
   - **Name** — a short, URL-friendly identifier; this becomes part of the generated API's base path (e.g. `inventory_db`).
   - **Label** — a human-friendly display name (admin UI only).
   - **Description** — optional notes for other admins (admin UI only).

## Step 4: Point it at your database file

1. Switch to the **Config** tab.
2. Enter the full path to your SQLite file in the database file/path field.
3. Adjust any additional connection options if present (e.g. caching settings).
4. Click **Save**.

DreamFactory connects to the file, introspects the schema (tables, columns, relationships), and immediately generates REST endpoints for every table it finds.

## Step 5: Review the auto-generated REST API

1. Open **API Docs** for your new service to see the live, interactive Swagger/OpenAPI documentation.
2. You'll see standard endpoints per table, typically supporting:
   - `GET /_table/{table_name}` — list/query records
   - `GET /_table/{table_name}/{id}` — fetch a single record
   - `POST /_table/{table_name}` — create record(s)
   - `PATCH /_table/{table_name}/{id}` — update a record
   - `DELETE /_table/{table_name}/{id}` — delete a record
3. Use **Try it out** in API Docs to test a call directly from the browser.

**Note:** unlike some other database connectors, SQLite doesn't support stored procedures or functions, so you won't see `_proc` endpoints for this service.

## Step 6: Secure the API with roles

APIs are not publicly usable until you grant access:

1. Go to **Roles & Access** and create a new role.
2. Under that role's **Access** tab, grant permissions (GET/POST/PATCH/DELETE) on specific tables in your SQLite service — you don't have to expose every table or verb.
3. Create an **app** and generate an **API key** tied to that role, and/or configure JWT-based user auth if you need per-user access control.

## Step 7 (optional): Add custom logic with scripting

If you need validation, computed fields, or side effects (e.g. sending a notification on insert), attach a server-side script to a table's pre- or post-process events:

- Supported languages typically include **Node.js, PHP, and Python**, depending on which script engines are installed.
- Scripts are attached under the service's **Scripts** tab at the table/event level.

## Step 8: Call your new REST API

```
GET https://your-dreamfactory-instance/api/v2/inventory_db/_table/products
```

with your API key in the `X-DreamFactory-API-Key` header (and a session token if using role/user-based auth). Your client apps now talk to a governed, documented REST API instead of opening a direct connection to the SQLite file.

## Notes

- Because SQLite is a single-file, embedded database, it's best suited for development, testing, or small/embedded deployments — for concurrent write-heavy production workloads, consider a client-server database instead.
- Back up the `.db` file before testing write operations (POST/PATCH/DELETE) against it.
