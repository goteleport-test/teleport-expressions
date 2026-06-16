# teleport-expressions

In-browser evaluators for [Teleport](https://goteleport.com)'s predicate
languages, built from Teleport's own parser code compiled to WebAssembly. Each
evaluator runs entirely in the browser.

Two evaluators are served:

- **`/labels`** - label expressions, the predicate used in Teleport role label
  matching, for example
  `labels["env"] == "prod" && contains(user.spec.traits["groups"], labels["owner"])`.
- **`/app-access`** - app-access resource rules, which match an HTTP request
  against a path pattern, an optional method, and an identity condition over
  the path segments a rule's captures bind.

The index page at `/` links to both.

## Layout

- `labelexpr/` - the label expression evaluator and its `Evaluate` function.
- `appaccess/` - the app-access rule evaluator and its `Evaluate` function.
- `internal/typical/` - Teleport's `typical` predicate library, copied verbatim.
- `internal/set/` - Teleport's `set` package, copied verbatim.
- `internal/resourcematcher/` - Teleport's app-access rule engine, copied from
  upstream with small demo-only additions (see Attribution).
- `cmd/eval/` - command-line label expression evaluator.
- `cmd/labels-wasm/`, `cmd/appaccess-wasm/` - WebAssembly entry points.
- `cmd/serve/` - tiny static file server for the web pages.
- `web/` - the browser front end, one directory per evaluator plus a shared
  stylesheet and the index page.

## Running locally

```sh
make serve
# open http://localhost:8080
```

`make serve` builds both WebAssembly modules and serves `web/` at
`http://localhost:8080`, with the evaluators at `/labels` and `/app-access`.

## Command line

The label expression evaluator is also available as a command:

```sh
make build
echo 'labels: {env: prod, owner: devs}
username: alice@example.com
traits: {groups: [devs, security]}' |
  ./bin/eval -expr 'contains(user.spec.traits["groups"], labels["owner"])'
# true
```

The input is YAML with three optional top-level keys: `labels` (a string map),
`username` (a string), and `traits` (a map of string lists).

## Label expression language

The label evaluator supports the same variables and functions as Teleport's
label expressions: the `labels` map, `user.metadata.name`, `user.spec.traits`,
and the functions `set`, `contains`, `contains_any`, `contains_all`,
`labels_matching`, `regexp.match`, `regexp.replace`, `email.local`,
`strings.upper`, and `strings.lower`.

## App-access rules

The app-access evaluator takes an `app_resources` list, the field a role
carries alongside `app_labels`. The rules are additive: a request is allowed if
any rule matches. Each rule uses either the declarative fields (`paths`,
`methods`, `where`) or a single bare `pred`, never both. A path pattern maps
`{name}` to a capture, `*` to a single segment, and `**` to a trailing greedy
match. A `where` or `pred` condition reads the request (`request.method`,
`request.path`), the identity (`user.name`, `user.roles`, `user.traits`), and
the captures a match bound (`vars.<name>`). The input is YAML holding a
`request` (`method`, `path`) and an `identity` (`name`, `roles`, `traits`).

A rule may also configure URL decoding and carry decision metadata.
`url_decoding` decodes the request path a set number of passes
(`decode_iterations`) before matching, then `allow_percent` decides whether a
residual `%` is admitted. On an allow a rule can return an `allow_code` and
`allow_reason`. On a deny each `deny_hint` entry (a `deny_code`, an optional
`deny_reason`, and an optional `on` condition that defaults to the path and
method clause) explains a near miss. The result panel shows the allow code and
reason on an allow, and every deny hint that fired on a deny.

The page's sugared checkbox toggles each example between its declarative form
and the predicate it lowers to. The desugared form is computed live by the
evaluator, so it always matches what the declarative form compiles to.

## Deployment

A GitHub Actions workflow (`.github/workflows/pages.yml`) runs the tests, builds
both WebAssembly modules, and publishes `web/` to GitHub Pages on every push to
`main`.

## Attribution and license

The contents of `internal/typical`, `internal/set`, and the parser
specification and helper functions in `labelexpr/` are copied, with light edits,
from [gravitational/teleport](https://github.com/gravitational/teleport) at tag
`v18.8.3`. The relevant source files are
[`lib/services/label_expressions.go`](https://github.com/gravitational/teleport/blob/v18.8.3/lib/services/label_expressions.go),
`lib/utils/typical/`, `lib/utils/set/`, `lib/utils/replace.go`, and
`lib/utils/parse/parse.go`.

The contents of `internal/resourcematcher` are copied from the Teleport branch
`julia/app/policy-matcher-sketch` (`lib/srv/app/resourcematcher`), with the
`typical` import path rewritten and two demo-only changes. A `DesugaredRule`
type lets the web front end render the lowered form of a rule, and the
`desugar()` function wraps the `where` clause in parentheses only when it is
combined with a path or method clause, rather than unconditionally.

Because it incorporates Teleport source, this project is licensed under the GNU
Affero General Public License v3.0. See [LICENSE](LICENSE).
