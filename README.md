# web

A mini web framework for [Caju](https://github.com/aecdanjundex/cajulang), in the style of Go's
[Gin](https://gin-gonic.com), for building REST APIs: routes with parameters, groups, middleware,
JSON input and output, and tests without a network.

```caju
import "github.com/aecdanjundex/web"

struct User {
  name: str
  age: int
}

r := web.default_engine()        # comes with logger() and recovery()

r.get("/users/:id", fn(c: *web.Context) {
  c.json(200, map[str]any{"id": c.param("id"), "page": c.default_query("page", "1")})
})

r.post("/users", fn(c: *web.Context) {
  u := &User{}
  if c.bind(u) != nil { return }   # invalid JSON: already responded 400 {"error": ...}
  c.json(201, *u)
})

api := r.group("/api")
api.use(web.basic_auth(map[str]str{"admin": "s3cret"}))
api.get("/stats", fn(c: *web.Context) { c.json(200, map[str]any{"user": c.must_get("user")}) })

r.run(":8080")
```

## Installation

Requires Caju 0.3.0 or newer (std/json and the HTTP server with keep-alive).

```sh
caju pkg init                          # if the project has no caju.toml yet
caju pkg add github.com/aecdanjundex/web
caju run main.caju
```

Full example, a to-do API (in-memory CRUD): [`examples/todo_api.caju`](examples/todo_api.caju).

## API

### Engine and groups

| | |
|---|---|
| `web.new()` | engine with no middleware |
| `web.default_engine()` | engine with `web.logger()` and `web.recovery()` (the `gin.Default()`; `default` is a reserved word in Caju) |
| `r.get/post/put/patch/delete/options/head(path, h)` | registers a route; `h` is `fn(c: *web.Context)` |
| `r.any(path, h)` | the same handler for every method |
| `r.handle(method, path, h)` | any method |
| `r.route(method, path, []fn(*web.Context){mw1, mw2, h})` | a route with its own middleware (Caju has no variadic parameters) |
| `r.use(mw)` | middleware for every request (including 404 and 405) |
| `g := r.group("/api")`, `g.group("/v1")`, `g.use(mw)` | nested groups, with the same route methods |
| `r.no_route(h)`, `r.no_method(h)` | 404 and 405 handlers |
| `r.redirect_trailing_slash`, `r.handle_method_not_allowed` | options (both `true` by default) |
| `r.run(":8080")` / `r.run("127.0.0.1:8080")` | serves forever in one process; returns an `error` only on failure |
| `r.run_workers(":8080", n)` | the same in `n` processes sharing the port (`n <= 0`: one per CPU); **each process has its own memory** |
| `r.serve(req, res)` | handles an `*http.Request` from `std/http` |
| `web.test_request(r, method, target, body, headers)` | runs a request without a network and returns a `*web.TestResponse` (`status`, `headers`, `body`, `header(name)`) |

**Routes:** `/users/:id` captures one segment and `/files/*path` captures the rest (with the leading
slash, as in Gin: `/files/a/b` → `"/a/b"`). Static segments take precedence over `:param`, which
takes precedence over `*catch-all`. If the static route only exists with another method, the
`:param` route handles the request. Values come decoded (`%20` → space).

**Automatic responses:**
- No route: 404 `{"error":"not found"}`.
- The path exists with other methods: 405 `{"error":"method not allowed"}` with the `Allow` header.
- Trailing slash: as in Gin, `/users/` redirects to `/users` (and vice versa). The code is 301 for
  GET and HEAD and 307 for other methods, and the query is kept.
- `HEAD` uses the `GET` route, without a body.

**Middleware order:** the engine's (`r.use`), then each group's, from the outermost to the
innermost, then the route's (`r.route`). Unlike Gin, a `use()` called after a route is registered
also applies to that route.

### Context

| request | |
|---|---|
| `c.param("id")` | route parameter |
| `c.query("q")`, `c.default_query("page", "1")`, `c.get_query("q")` → `(str, bool)`, `c.query_array("tag")` | query string, decoded |
| `c.header("X-Token")`, `c.method()`, `c.path()`, `c.full_path()` (the pattern: `/users/:id`), `c.body()` | |
| `c.client_ip()` | from `X-Forwarded-For`, `X-Real-Ip` or, without them, the connection address |
| `c.bind(&v)` | reads the JSON body into `v`; on error, responds 400 `{"error": ...}`, aborts and returns the error |
| `c.should_bind_json(&v)` | the same without responding: the handler decides |

| response | |
|---|---|
| `c.json(code, v)`, `c.indented_json(code, v)` | any value: maps, lists, structs, numbers, text |
| `c.string(code, text)`, `c.data(code, content_type, body)` | |
| `c.status(code)`, `c.set_header(name, value)`, `c.redirect(code, url)` | |

| flow | |
|---|---|
| `c.next()` | runs the rest of the chain and comes back (to measure, wrap, post-process) |
| `c.abort()`, `c.abort_with_status(code)`, `c.abort_with_status_json(code, v)`, `c.is_aborted()` | stops the chain |
| `c.set(key, v)`, `c.get(key)` → `(any, bool)`, `c.must_get(key)` | per-request values shared between middleware |

There is no `gin.H`, because Caju has no type aliases. Use `map[str]any{"chave": valor}`.

### Built-in middleware

| | |
|---|---|
| `web.logger()` | one line per request: `[WEB] 200 \|   153.2µs \| - \|     GET "/users/1"` |
| `web.logger_to(fn(line: str) {...})` | the same, writing wherever you want |
| `web.recovery()` | a `panic` becomes 500 `{"error":"internal server error"}` (the message goes to stderr) and the server keeps running |
| `web.cors("*")` | CORS headers; responds 204 to `OPTIONS` (preflight) requests |
| `web.basic_auth(map[str]str{"user": "senha"})` | HTTP basic authentication; stores the user in `c.get("user")` |

## Tests

```caju
import { describe, it, expect } from "std/testing"
import "github.com/aecdanjundex/web"

describe("GET /users/:id", fn() {
  it("devolve o id", fn() {
    res := web.test_request(app(), "GET", "/users/7", "", nil)
    expect(res.status).to_equal(200)
    expect(res.body).to_equal("{\"id\":\"7\"}")
  })
})
```

To run the package's own tests: `caju test`.

## Differences from Gin

- The `std/http` server is an event loop (epoll) in one process: it keeps many connections open at
  once, with keep-alive and pipelining, but **runs one handler at a time** (there are no
  goroutines). A slow handler holds up the other clients of that process.
- To use several cores, `r.run_workers(addr, n)` starts `n` processes on the same port
  (`SO_REUSEPORT`). They **do not share memory**: a global map changed by a handler exists
  separately in each process. Use it with stateless handlers or with state kept outside the process.
- No variadic parameters: route middleware goes in a list, in `r.route(...)`.
- `web.default_engine()` instead of `Default()`, and `map[str]any` instead of `gin.H`.
- No HTML templates, multipart uploads, forms (`c.post_form`), static files or HTTPS.

## License

MIT
