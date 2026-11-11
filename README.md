# REST API Graph – Traefik plugin

A [Traefik](https://traefik.io) middleware plugin that lets clients shape the
responses of an existing JSON REST API, the way they would with a graph API:
pick the fields they need, pull linked resources into the response, and
send several requests at once. The backend does not have to change.

| Feature | Request |
|---------|---------|
| [Field filtering](#field-filtering) | `GET /groups?fields=id,users.name` or `?fields=-title,-users.rel` |
| [Inclusion](#inclusion) | `GET /groups?include=users,users.friends(limit:1)` |
| [Batch](#batch) | `POST /batch` with `{"a":"/groups","b":"/users"}` |

## Inspiration

This plugin is a Go port of
[**Rainbow REST**](https://github.com/alexeytokar/rainbow-rest), a set of
Java servlet filters "inspired by JsonApi, designed to replace GraphQL".
Rainbow REST adds the same features to a Java application from the inside;
this plugin applies them at the proxy, so they work for an API written in any
language.

The request syntax and the overall behaviour come from Rainbow REST: its
`RainbowRestWebFilter` (fields and include) and `RainbowRestBatchFilter`
(batch) are combined here in a single middleware. The places where the plugin
intentionally behaves differently are listed under
[Differences from Rainbow REST](#differences-from-rainbow-rest).

## Behaviour

The examples use a backend answering `GET /groups` with:

```json
[
  {"id": 2, "title": "bad group",  "users": {"href": "/groups/2/users", "rel": "users"}},
  {"id": 5, "title": "good group", "users": {"href": "/groups/5/users", "rel": "users"}}
]
```

and `GET /groups/2/users` with:

```json
[
  {"name": "boss", "friends": {"href": "/users/1/friends", "rel": "friends"}},
  {"name": "cat",  "friends": {"href": "/users/2/friends", "rel": "friends"}}
]
```

Requests without `fields` and `include` are passed through untouched, and so
are responses that are not JSON.

### Inclusion

A *link* is a JSON object with an `href` field. `include` replaces links with
the resources they point to, fetched with a GET request.

`GET /groups?include=users`

```json
[
  {"id": 2, "title": "bad group", "users": [
    {"name": "boss", "friends": {"href": "/users/1/friends", "rel": "friends"}},
    {"name": "cat",  "friends": {"href": "/users/2/friends", "rel": "friends"}}
  ]},
  {"id": 5, "title": "good group", "users": [ ... ]}
]
```

* `include` is a comma separated list of dot separated paths, relative to the
  top level of the response. Arrays are transparent: `users` above addresses
  the `users` field of every group.
* Nested links are resolved by listing the nested path as well:
  `include=users,users.friends`. Parents are resolved before their children.
* Query parameters for the link can be given in parentheses:
  `include=users(offset:1,limit:2)` requests `/groups/2/users?offset=1&limit=2`.
  Entries that are not a `name:value` pair are ignored.
* A field holding an array of links is resolved element by element.
* Paths that do not lead to a link are ignored.
* All the links of one include are fetched in parallel.

### Field filtering

`fields` is a comma separated list of the same dot separated paths. A path
without a prefix selects a field, a path prefixed with `-` removes it.

| `fields=` | Result |
|-----------|--------|
| `id` | `[{"id":2},{"id":5}]` |
| `-title` | Everything but the top level `title`. |
| `-users.rel` | Everything but the `rel` of `users`. |
| `id,users` | Only `id` and `users`, the latter with all of its fields. |
| `users.name` | Only `users`, and of those only `name`. |
| `id,users,-users.rel` | Only `id` and `users`, the latter without `rel`. |

* A name only matches on the level its path points to: `-title` removes the
  top level `title` and leaves `users.title` alone, `-users.title` does the
  opposite.
* Selecting a field restricts the level it is on to the selected fields.
  Levels without a selected field keep everything that is not removed.
* When a field is both selected and removed, the removal wins.
* Arrays are transparent here too.

`include` and `fields` can be combined. Includes are resolved first, so
`fields` can address what was included:

`GET /groups?include=users&fields=title,users.name`

```json
[
  {"title": "bad group",  "users": [{"name": "boss"}, {"name": "cat"}]},
  {"title": "good group", "users": [ ... ]}
]
```

### Batch

`POST /batch` takes a JSON object mapping names to relative URLs, performs
all the GET requests in parallel and returns an object with the same names:

```sh
curl -X POST http://localhost:8000/batch \
  -d '{"groups": "/groups?fields=id", "users": "/groups/2/users?limit=1"}'
```

```json
{
  "groups": [{"id": 2}, {"id": 5}],
  "users": [{"name": "boss", "friends": {"href": "/users/1/friends", "rel": "friends"}}]
}
```

* The entries may use `include` and `fields` themselves.
* An entry that fails (unreachable, `5xx`, not JSON) becomes `null`; the
  other entries are still returned.
* A body that is not an object of strings, or that has more than
  `maxBatchSize` entries, is answered with `400 Bad Request`.

### Sub-requests

Links and batch entries are resolved with GET sub-requests.

* The headers of the original request (authorization, cookies, ...) are
  forwarded, except `Content-Length`, `Accept-Encoding`, `Range` and the
  conditional `If-*` headers.
* Only the path and the query of a link are used. Its scheme and host are
  ignored, so a response can never make the plugin call another server.
* All the sub-requests of one request share a time limit
  (`executionTimeoutSeconds`).
* Links that keep including themselves are cut off after 16 levels.

There are two ways to perform them:

* **In-process (default).** The sub-request is handed to the rest of the
  middleware chain of the router the plugin is attached to. There is no
  extra network hop and nothing to configure, but links can only reach what
  that router's service can serve. These sub-requests are not separate
  entries in Traefik's access log, metrics and traces: they get a context of
  their own, which only shares the deadline and the cancellation of the
  original request.
* **`serverUri`.** When set (for instance to a Traefik entry point,
  `http://localhost:80`), sub-requests are real HTTP requests to that
  server. Links may then cross routers and services.

### Errors and edge cases

| Situation | Response |
|-----------|----------|
| Response is not JSON, is compressed, or is empty | Passed through unchanged. |
| A link cannot be resolved (request fails, `5xx`, not JSON) | `502 Bad Gateway` |
| The sub-requests exceed the time limit | `504 Gateway Timeout` |
| Invalid batch body or too many entries | `400 Bad Request` |

Errors produced by the plugin have a `{"message": "..."}` body.

Whenever `include` or `fields` is used, the upstream is asked for an
uncompressed response. Put Traefik's `compress` middleware *before* this
plugin to compress the transformed response. `include` and `fields` are read
from the query string only.

## Configuration

| Option | Default | Description |
|--------|---------|-------------|
| `fieldsParamName` | `fields` | Query parameter for field filtering. Empty disables the feature. |
| `includeParamName` | `include` | Query parameter for inclusion. Empty disables the feature. |
| `batchEndpointUri` | `/batch` | Path of the batch endpoint (matched case-insensitively, `POST` only). Empty disables the feature. |
| `maxBatchSize` | `0` | Maximum number of entries in a batch request, `0` means unlimited. |
| `executionTimeoutSeconds` | `10` | Time limit for the sub-requests of one request. |
| `serverUri` | _(empty)_ | Server to send sub-requests to, see [Sub-requests](#sub-requests). |

```yaml
# dynamic configuration
http:
  routers:
    api:
      rule: PathPrefix(`/`)
      service: api
      middlewares: [rest-api-graph]

  middlewares:
    rest-api-graph:
      plugin:
        restapigraph:
          maxBatchSize: 20
```

## Installation

The plugin is not published to the Traefik Plugin Catalog yet. Use it as a
[local plugin](https://plugin-catalog.traefik.io/install-plugin): make a
checkout of this repository available to Traefik as
`./plugins-local/src/github.com/alexeytokar/traefik-rest-api-graph` and
declare it in the static configuration:

```yaml
experimental:
  localPlugins:
    restapigraph:
      moduleName: github.com/alexeytokar/traefik-rest-api-graph
```

### Demo

[`demo/`](demo) contains a ready to run setup: Traefik, the plugin and a
small JSON backend mirroring the sample application of Rainbow REST.

```sh
cd demo
docker compose up

curl 'http://localhost:8000/groups?fields=id'
curl 'http://localhost:8000/groups?include=users(limit:1),users.friends&fields=title,users.name,users.friends.name'
curl -X POST -d '{"groups":"/groups","users":"/groups/2/users"}' http://localhost:8000/batch
```

## Differences from Rainbow REST

* `fields` takes paths. Rainbow REST matches a field name on every level of
  the response instead, so `fields=-title` there also removes `users.title`.
* Responses that are not JSON are passed through unchanged instead of
  failing the request.
* A link that cannot be resolved results in `502 Bad Gateway` and a timeout
  in `504 Gateway Timeout`. A sub-response with a `5xx` status counts as a
  failure instead of being embedded.
* An `include` path that does not lead to a link is ignored instead of
  raising an error.
* Self-including links are cut off after 16 levels.
* Sub-requests are handled in-process by default. Rainbow REST always calls
  its own server over HTTP, which is what `serverUri` does here.
* `include` and `fields` are read from the query string only, not from form
  encoded bodies.
* Both filters are one middleware; a feature is turned off by setting its
  option to an empty value.

## Development

The plugin has no dependencies besides the Go standard library, as required
by [Yaegi](https://github.com/traefik/yaegi), the interpreter Traefik runs
plugins with.

```sh
go test -race ./...
```

The test suite also passes when interpreted (`yaegi test .` from a `GOPATH`
checkout), and the plugin has been load tested inside Traefik v3.2 as a
local plugin, with access log, metrics and tracing enabled.

Yaegi is less forgiving than the Go compiler when it comes to concurrency:
`interface{}` based JSON trees, goroutines started from closures and
`select` all misbehaved under load inside Traefik while working fine when
compiled. The code therefore uses a concrete JSON node type and starts
sub-request goroutines without closures or `select`; please keep it that
way and load test inside a real Traefik when touching the concurrent parts.

## License

[MIT](LICENSE), like Rainbow REST.
