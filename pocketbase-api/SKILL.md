---
name: pocketbase-api
description: Expert-level PocketBase v0.36.x skill covering JavaScript (JSVM pb_hooks) and Go extension APIs, plus the REST API and JS SDK. Use for PocketBase hooks, custom routes, record CRUD, collections, database queries, cron jobs, realtime/SSE, file handling, migrations, HTTP requests from hooks, auth/token management, API rules, or the JS SDK. Also trigger on mentions of pb_hooks, $app, e.app, routerAdd, onRecord*, cronAdd, or pb.collection().
---

# PocketBase Expert Reference

PocketBase v0.36.x — open source backend: embedded SQLite, realtime SSE, auth, dashboard, REST API.
Extensible via **JavaScript** (JSVM, `pb_hooks/*.js`) or **Go** (use as framework).

## Quick Reference: When to Use What

- **`$app`** — use in cron callbacks and top-level hook scope
- **`e.app`** — use inside route handlers and event hooks (the request-scoped app instance)
- **`txApp`** — use inside `runInTransaction()` callbacks (NEVER use `$app` inside transactions)
- **JS hooks location**: `pb_hooks/*.js` (auto-loaded on startup)
- **Go hooks**: registered via `app.OnServe().BindFunc(...)` etc.

---

## JS Extension API

### Routing

Register custom routes with `routerAdd(method, path, handler, ...middlewares)`:

```js
routerAdd("GET", "/hello/{name}", (e) => {
  let name = e.request.pathValue("name")
  return e.json(200, { "message": "Hello " + name })
})

routerAdd("POST", "/api/myapp/settings", (e) => {
  return e.json(200, {"success": true})
}, $apis.requireAuth())
```

Path patterns follow Go `net/http.ServeMux` rules:
- `{paramName}` — single segment
- `{paramName...}` — wildcard (multiple segments)
- Trailing `/` — anonymous wildcard
- `{$}` — exact match with trailing slash

Prefix custom routes under `/api/myapp/...` to avoid collisions with system routes.

#### Request Event (`e`)

```js
e.request.pathValue("id")              // path param
e.request.url.query().get("search")    // query param
e.request.header.get("Some-Header")    // request header
e.response.header().set("X-Custom", "val")  // response header

e.auth                    // current auth record (null if guest)
e.hasSuperuserAuth()      // check if superuser

let info = e.requestInfo()
info.body                 // parsed request body
info.query                // query params
info.headers              // normalized headers
info.auth                 // auth record

// Bind body to typed struct
const data = new DynamicModel({ title: "", active: false })
e.bindBody(data)

// Response methods
e.json(200, {...})
e.string(200, "text")
e.html(200, "<h1>Hi</h1>")
e.redirect(307, "https://...")
e.noContent(204)
e.fileFS($os.dirFS("..."), "file.txt")

// Request store (share data between middlewares)
e.set("key", value)
e.get("key")
```

#### Middlewares

```js
// Global
routerUse((e) => {
  // before
  return e.next()  // MUST call e.next() to continue chain
})

// With custom priority
routerUse(new Middleware((e) => {
  return e.next()
}, -1))  // negative = runs earlier

// Route-specific (pass after handler)
routerAdd("GET", "/path", handler, middlewareFunc)
```

**Builtin middlewares** (`$apis.*`):
- `$apis.requireAuth(optCollectionNames...)` — authenticated users only
- `$apis.requireSuperuserAuth()` — superusers only
- `$apis.requireGuestOnly()` — guests only
- `$apis.requireSuperuserOrOwnerAuth(ownerIdParam)` — superuser or record owner
- `$apis.bodyLimit(bytes)` — custom body size limit (default 32MB)
- `$apis.gzip()` — gzip compression
- `$apis.skipSuccessActivityLog()` — log only errors

#### Error Handling

```js
throw new BadRequestError(optMessage, optData)   // 400
throw new UnauthorizedError(optMessage, optData)  // 401
throw new ForbiddenError(optMessage, optData)     // 403
throw new NotFoundError(optMessage, optData)      // 404
throw new TooManyrequestsError(optMessage, optData) // 429
throw new InternalServerError(optMessage, optData)  // 500

// With validation data
throw new ApiError(500, "something went wrong", {
  "title": new ValidationError("invalid_title", "Invalid or missing title"),
})
```

### Record Operations

#### Fetch Records

```js
// Single record
$app.findRecordById("articles", "RECORD_ID")
$app.findFirstRecordByData("articles", "slug", "test")
$app.findFirstRecordByFilter("articles",
  "status = 'public' && category = {:category}",
  { "category": "news" }
)

// Multiple records
$app.findRecordsByIds("articles", ["ID1", "ID2"])
$app.findRecordsByFilter("articles",
  "status = 'public'",  // filter
  "-created",           // sort
  10,                   // limit
  0,                    // offset
  { "category": "news" } // optional params
)
$app.findAllRecords("articles",
  $dbx.hashExp({"status": "pending"})
)
$app.countRecords("articles", $dbx.hashExp({"status": "pending"}))

// Auth records
$app.findAuthRecordByEmail("users", "test@example.com")
$app.findAuthRecordByToken("TOKEN", "auth")
```

**Always use `{:placeholder}` for untrusted user input in filter expressions.**

#### Create / Update / Delete

```js
// Create
let collection = $app.findCollectionByNameOrId("articles")
let record = new Record(collection)
record.set("title", "Lorem ipsum")
record.set("active", true)
record.set("slug:autogenerate", "post-")  // field modifier
$app.save(record)

// File upload
let f = $filesystem.fileFromPath("/path/to/file.txt")
record.set("documents", [f])

// Update
let record = $app.findRecordById("articles", "ID")
record.set("title", "Updated")
record.set("documents-", ["old_file.txt"])  // remove file
record.set("documents+", [newFile])          // append file
$app.save(record)

// Delete
$app.delete(record)
```

#### Record Field Access

```js
record.get("field")              // any
record.getString("field")        // string
record.getInt("field")           // int
record.getBool("field")          // bool
record.getFloat("field")         // float
record.getDateTime("field")      // DateTime
record.getStringSlice("field")   // []string

record.set("field", value)
record.set("field+", appendValue)   // append
record.set("field-", removeValue)   // remove

record.expandedOne("author")        // single expanded relation
record.expandedAll("categories")    // multiple expanded relations

record.original()  // shallow copy with original DB state
record.fresh()     // shallow copy with latest state, no expand
record.clone()     // full shallow copy

// Auth record helpers
record.email()
record.setEmail(email)
record.verified()
record.validatePassword(pass)
record.setPassword(pass)
record.newAuthToken()
record.newFileToken()
```

#### Expand Relations

```js
$app.expandRecord(record, ["author", "categories"], null)
$app.expandRecords(records, ["author"], null)
```

### Event Hooks

All hooks: `function(e) { ... e.next() }`. Throwing or not calling `e.next()` stops the chain.

**Record model hooks** (triggered from anywhere — no request context):
```js
onRecordCreate((e) => { /* e.record */ e.next() }, "collection_name")
onRecordCreateExecute((e) => { e.next() }, "collection_name")
onRecordAfterCreateSuccess((e) => { e.next() }, "collection_name")
onRecordAfterCreateError((e) => { /* e.error */ e.next() }, "collection_name")

onRecordUpdate((e) => { e.next() }, "collection_name")
onRecordUpdateExecute((e) => { e.next() }, "collection_name")
onRecordAfterUpdateSuccess((e) => { e.next() }, "collection_name")

onRecordDelete((e) => { e.next() }, "collection_name")
onRecordDeleteExecute((e) => { e.next() }, "collection_name")
onRecordAfterDeleteSuccess((e) => { e.next() }, "collection_name")

onRecordValidate((e) => { e.next() }, "collection_name")
onRecordEnrich((e) => { e.record.hide("field"); e.next() }, "collection_name")
```

**Record request hooks** (have request context):
```js
onRecordCreateRequest((e) => {
  e.record.set("status", "pending")
  e.next()
}, "articles")

onRecordUpdateRequest((e) => { e.next() }, "articles")
```

**Hook execution order** for `$app.save()`:
`onRecordCreate` → `onRecordValidate` → `onRecordCreateExecute` → (commit) → `onRecordAfterCreateSuccess`

**`AfterSuccess` hooks** are delayed until transaction commit. Use them for side effects that should only happen on actual persistence.

### Database (Raw SQL)

```js
// Raw query
$app.db().newQuery("DELETE FROM articles WHERE status = 'archived'").execute()

// Select single row
const result = new DynamicModel({ id: "", status: false })
$app.db().newQuery("SELECT id, status FROM users WHERE id=1").one(result)

// Select multiple rows
const results = arrayOf(new DynamicModel({ id: "", email: "" }))
$app.db().newQuery("SELECT id, email FROM users LIMIT 100").all(results)

// Parameterized queries (ALWAYS use for user input)
$app.db().newQuery("SELECT * FROM posts WHERE created >= {:from}")
  .bind({ "from": "2023-06-25 00:00:00.000Z" })
  .all(results)
```

**Query builder**:
```js
$app.db()
  .select("id", "email")
  .from("users")
  .andWhere($dbx.like("email", "example.com"))
  .andWhere($dbx.hashExp({ status: "active" }))
  .orderBy("created ASC")
  .limit(100)
  .offset(0)
  .all(results)
```

**dbx expressions**:
- `$dbx.exp(raw, params)` — raw expression with bindings
- `$dbx.hashExp({key: value})` — equality map (supports null, arrays, booleans)
- `$dbx.not(exp)`, `$dbx.and(...exps)`, `$dbx.or(...exps)`
- `$dbx.in(col, ...vals)`, `$dbx.notIn(col, ...vals)`
- `$dbx.like(col, ...vals)`, `$dbx.notLike(col, ...vals)`
- `$dbx.between(col, from, to)`, `$dbx.notBetween(col, from, to)`
- `$dbx.exists(exp)`, `$dbx.notExists(exp)`

### Transactions

```js
$app.runInTransaction((txApp) => {
  // ALWAYS use txApp inside, NEVER $app (deadlock risk)
  let record = txApp.findRecordById("articles", "ID")
  record.set("status", "active")
  txApp.save(record)
})
```

### Cron Jobs

```js
cronAdd("jobId", "*/5 * * * *", () => {
  // runs every 5 minutes
  // Use $app here (NOT e.app — no request context)
  console.log("tick")
})

cronRemove("jobId")
```

Cron expression: `minute hour day-of-month month day-of-week`. Supports macros like `@hourly`, `@daily`.

### HTTP Requests (from hooks)

```js
const res = $http.send({
  url: "https://api.example.com/data",
  method: "POST",
  body: JSON.stringify({ key: "value" }),
  headers: { "content-type": "application/json" },
  timeout: 120,  // seconds
})
res.statusCode   // HTTP status
res.json         // parsed JSON
res.body         // raw bytes
res.headers      // response headers
res.cookies      // response cookies
```

**Multipart/form-data**:
```js
const formData = new FormData()
formData.append("title", "Hello")
formData.append("file", $filesystem.fileFromBytes("content", "file.txt"))
$http.send({ url: "...", method: "POST", body: formData })
```

Limitation: No streaming/SSE support in `$http.send` — use Go for that.

### Auth Helpers

```js
$apis.recordAuthResponse(e, record, "provider")  // standard auth response (token + record)
$app.canAccessRecord(record, e.requestInfo(), record.collection().viewRule)
```

---

## Go Extension API

### Routing

```go
app.OnServe().BindFunc(func(se *core.ServeEvent) error {
    se.Router.GET("/hello/{name}", func(e *core.RequestEvent) error {
        name := e.Request.PathValue("name")
        return e.JSON(http.StatusOK, map[string]any{"name": name})
    })

    // With middleware
    se.Router.POST("/api/myapp/action", handler).Bind(apis.RequireAuth())

    // Groups
    g := se.Router.Group("/api/myapp")
    g.Bind(apis.RequireAuth())
    g.GET("/items", listHandler)
    g.POST("/items", createHandler)

    return se.Next()
})
```

Request/response methods are identical to JS but with Go capitalization:
`e.Request.PathValue()`, `e.JSON()`, `e.String()`, `e.Auth`, `e.HasSuperuserAuth()`, `e.BindBody(&data)`.

### Go Hooks

Same hook names as JS but registered via `app.OnRecord*().BindFunc(...)`:

```go
app.OnRecordCreate("articles").BindFunc(func(e *core.RecordEvent) error {
    e.Record.Set("status", "pending")
    return e.Next()
})
```

---

## REST API (Client-Side)

Base: `http://127.0.0.1:8090/api/`

### Records CRUD

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/collections/{name}/records` | List/search (paginated). Params: `page`, `perPage`, `sort`, `filter`, `expand`, `fields`, `skipTotal` |
| GET | `/api/collections/{name}/records/{id}` | View single. Params: `expand`, `fields` |
| POST | `/api/collections/{name}/records` | Create. Body: JSON or multipart |
| PATCH | `/api/collections/{name}/records/{id}` | Update |
| DELETE | `/api/collections/{name}/records/{id}` | Delete |

**Filter syntax**: `field operator value`, combined with `&&` (AND), `||` (OR), `(...)`.
Operators: `=`, `!=`, `>`, `>=`, `<`, `<=`, `~` (like), `!~` (not like), `?=` (any equal), etc.

**Sort**: prefix `-` for DESC, `+` (default) for ASC. E.g. `?sort=-created,id`.

**Expand**: `?expand=relField1,relField2.subRelField` (up to 6 levels).

### Realtime (SSE)

```js
// JS SDK
pb.collection('example').subscribe('*', (e) => {
  console.log(e.action)  // "create" | "update" | "delete"
  console.log(e.record)
})

pb.collection('example').subscribe('RECORD_ID', callback)

pb.collection('example').unsubscribe('*')
pb.collection('example').unsubscribe()  // all subscriptions
```

SSE endpoint: `GET /api/realtime` (establishes connection).
Subscriptions: `POST /api/realtime` with `clientId` + `subscriptions[]`.

Access rules: single record subscription uses `ViewRule`; collection subscription uses `ListRule`.
Auto-disconnect after 5 min idle (auto-reconnects if client is active).

### Auth

```js
// JS SDK
await pb.collection('users').authWithPassword('email', 'password')
await pb.collection('users').authRefresh()
pb.authStore.token  // current JWT
pb.authStore.record // current auth record
```

### Custom Routes (from SDK)

```js
await pb.send("/api/myapp/custom", {
  method: "POST",
  body: { key: "value" },
  query: { abc: "123" },
})
```

---

## Common Gotchas

1. **`$app` vs `e.app`**: Use `$app` only in `cronAdd` callbacks. Use `e.app` in route handlers and hooks.
2. **Transactions**: Always use `txApp` inside `runInTransaction()`. Using `$app` causes deadlock.
3. **`e.next()`**: Must be called in hooks/middlewares to continue the chain. Forgetting it silently stops execution.
4. **Parameterized queries**: Always use `{:placeholder}` syntax for user input to prevent SQL injection.
5. **Hook ordering**: `onRecordCreate` fires BEFORE validation/INSERT. `onRecordAfterCreateSuccess` fires AFTER commit.
6. **`DynamicModel` keys**: Cannot start with underscore, must be valid Go struct field names.
7. **`arrayOf()`**: Required when using `.all()` to populate multiple records.
8. **File fields**: Use `field+` to append, `field-` to remove by filename. New files must be `$filesystem.File` instances.
9. **Record request hooks vs model hooks**: Request hooks (`onRecordCreateRequest`) have access to request context. Model hooks (`onRecordCreate`) don't.
10. **`record.withCustomData(true)`**: Must be called explicitly to include custom (non-schema) fields in serialization.

## Full Docs Reference

- JS Types: https://pocketbase.io/jsvm/index.html
- Go pkg: https://pkg.go.dev/github.com/pocketbase/pocketbase
- Docs: https://pocketbase.io/docs/
