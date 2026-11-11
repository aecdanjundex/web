# web

Um mini framework web para [Caju](https://github.com/aecdanjundex/caju), no estilo do
[Gin](https://gin-gonic.com) do Go, para criar APIs REST: rotas com parâmetros, grupos,
middlewares, JSON de entrada e saída e testes sem rede.

```caju
import "github.com/aecdanjundex/web"

struct User {
  name: str
  age: int
}

r := web.default_engine()        # já vem com logger() e recovery()

r.get("/users/:id", fn(c: *web.Context) {
  c.json(200, map[str]any{"id": c.param("id"), "page": c.default_query("page", "1")})
})

r.post("/users", fn(c: *web.Context) {
  u := &User{}
  if c.bind(u) != nil { return }   # JSON inválido: já respondeu 400 {"error": ...}
  c.json(201, *u)
})

api := r.group("/api")
api.use(web.basic_auth(map[str]str{"admin": "s3cret"}))
api.get("/stats", fn(c: *web.Context) { c.json(200, map[str]any{"user": c.must_get("user")}) })

r.run(":8080")
```

## Instalação

Precisa da Caju 0.2.0 ou mais nova (que traz o `std/json`).

```sh
caju pkg init                          # se o projeto ainda não tem caju.toml
caju pkg add github.com/aecdanjundex/web
caju run main.caju
```

Exemplo completo, uma API de tarefas (CRUD em memória): [`examples/todo_api.caju`](examples/todo_api.caju).

## API

### Engine e grupos

| | |
|---|---|
| `web.new()` | engine sem middlewares |
| `web.default_engine()` | engine com `web.logger()` e `web.recovery()` (o `gin.Default()`; `default` é palavra reservada na Caju) |
| `r.get/post/put/patch/delete/options/head(path, h)` | registra uma rota; `h` é `fn(c: *web.Context)` |
| `r.any(path, h)` | o mesmo handler para todos os métodos |
| `r.handle(method, path, h)` | método qualquer |
| `r.route(method, path, []fn(*web.Context){mw1, mw2, h})` | rota com middlewares próprios (a Caju não tem parâmetros variádicos) |
| `r.use(mw)` | middleware de todas as requisições (inclusive 404 e 405) |
| `g := r.group("/api")`, `g.group("/v1")`, `g.use(mw)` | grupos aninhados, com os mesmos métodos de rota |
| `r.no_route(h)`, `r.no_method(h)` | handlers de 404 e 405 |
| `r.redirect_trailing_slash`, `r.handle_method_not_allowed` | opções (ambas `true` por padrão) |
| `r.run(":8080")` / `r.run("127.0.0.1:8080")` | serve para sempre num processo; devolve `error` só se falhar |
| `r.run_workers(":8080", n)` | o mesmo em `n` processos que dividem a porta (`n <= 0`: um por CPU); **cada processo tem sua própria memória** |
| `r.serve(req, res)` | atende um `*http.Request` do `std/http` |
| `web.test_request(r, method, target, body, headers)` | roda uma requisição sem rede e devolve `*web.TestResponse` (`status`, `headers`, `body`, `header(name)`) |

**Rotas:** `/users/:id` captura um segmento e `/files/*path` captura o resto (com a barra inicial,
como no Gin: `/files/a/b` → `"/a/b"`). Segmentos fixos têm prioridade sobre `:param`, que tem
prioridade sobre `*catch-all`. Se a rota fixa só existe com outro método, a de `:param` atende. Os
valores vêm decodificados (`%20` → espaço).

**Respostas automáticas:**
- Nenhuma rota: 404 `{"error":"not found"}`.
- O caminho existe com outros métodos: 405 `{"error":"method not allowed"}` com o cabeçalho `Allow`.
- Barra final: como no Gin, `/users/` redireciona para `/users` (e o contrário). O código é 301
  em GET e HEAD e 307 nos outros métodos, e a query é mantida.
- `HEAD` usa a rota `GET`, sem corpo.

**Ordem dos middlewares:** os do engine (`r.use`), depois os de cada grupo, do mais externo para o
mais interno, depois os da rota (`r.route`). Diferente do Gin, um `use()` feito depois de registrar
a rota também vale para ela.

### Context

| requisição | |
|---|---|
| `c.param("id")` | parâmetro da rota |
| `c.query("q")`, `c.default_query("page", "1")`, `c.get_query("q")` → `(str, bool)`, `c.query_array("tag")` | query string, decodificada |
| `c.header("X-Token")`, `c.method()`, `c.path()`, `c.full_path()` (o padrão: `/users/:id`), `c.body()` | |
| `c.client_ip()` | de `X-Forwarded-For`, `X-Real-Ip` ou, sem eles, o endereço da conexão |
| `c.bind(&v)` | lê o corpo JSON em `v`; com erro, responde 400 `{"error": ...}`, aborta e devolve o erro |
| `c.should_bind_json(&v)` | o mesmo sem responder: o handler decide |

| resposta | |
|---|---|
| `c.json(code, v)`, `c.indented_json(code, v)` | qualquer valor: mapas, listas, structs, números, texto |
| `c.string(code, text)`, `c.data(code, content_type, body)` | |
| `c.status(code)`, `c.set_header(name, value)`, `c.redirect(code, url)` | |

| fluxo | |
|---|---|
| `c.next()` | roda o resto da cadeia e volta (para medir, envolver, pós-processar) |
| `c.abort()`, `c.abort_with_status(code)`, `c.abort_with_status_json(code, v)`, `c.is_aborted()` | interrompe a cadeia |
| `c.set(key, v)`, `c.get(key)` → `(any, bool)`, `c.must_get(key)` | valores da requisição entre middlewares |

Não existe `gin.H`, porque a Caju não tem alias de tipo. Use `map[str]any{"chave": valor}`.

### Middlewares prontos

| | |
|---|---|
| `web.logger()` | uma linha por requisição: `[WEB] 200 \|   153.2µs \| - \|     GET "/users/1"` |
| `web.logger_to(fn(line: str) {...})` | o mesmo, escrevendo onde você quiser |
| `web.recovery()` | um `panic` vira 500 `{"error":"internal server error"}` (a mensagem vai para o stderr) e o servidor continua |
| `web.cors("*")` | cabeçalhos CORS; responde 204 às requisições `OPTIONS` (preflight) |
| `web.basic_auth(map[str]str{"user": "senha"})` | autenticação HTTP básica; guarda o usuário em `c.get("user")` |

## Testes

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

Para rodar os testes do próprio pacote: `caju test`.

## Diferenças em relação ao Gin

- O servidor do `std/http` é um laço de eventos (epoll) num processo: mantém muitas conexões
  abertas ao mesmo tempo, com keep-alive e pipelining, mas **roda um handler por vez** (não há
  goroutines). Um handler lento segura os outros clientes daquele processo.
- Para usar vários núcleos, `r.run_workers(addr, n)` sobe `n` processos na mesma porta
  (`SO_REUSEPORT`). Eles **não compartilham memória**: um mapa global alterado por um handler
  existe separado em cada processo. Use com handlers sem estado ou com o estado fora do processo.
- Sem parâmetros variádicos: middlewares de rota vão numa lista, em `r.route(...)`.
- `web.default_engine()` no lugar de `Default()`, e `map[str]any` no lugar de `gin.H`.
- Sem templates HTML, upload multipart, formulários (`c.post_form`), arquivos estáticos nem HTTPS.

## Licença

MIT
