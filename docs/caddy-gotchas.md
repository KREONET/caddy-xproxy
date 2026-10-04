# Caddy gotchas

[🇺🇸 English](caddy-gotchas.md) · [🇰🇷 한국어](caddy-gotchas.ko.md)

Why the configuration in this repo looks the way it does. All of it was measured on Caddy v2.11.4; none of it is guesswork.

## abort discards log_append

Block with `abort` and the log line survives (`status: 0`) but every field added by `log_append` is gone.

| approach | response | `block_reason` |
|--|--|--|
| `log_append` + `abort` | connection dropped | **lost** |
| `log_append` + `respond 403` | 403 | recorded |
| `log_append` + `error 403` | 403 | recorded |
| `log_append` + `respond 444` | 444 | recorded |
| tag with a matcher at site scope, `abort` elsewhere | connection dropped | **lost** |

The last row is the important one. Putting `log_append @matcher key value` at site scope and doing the `abort` in a different handle does not work around it either. If the reason has to reach the log, `abort` is not available.

That is why `00-probe.caddy` and `00-badbots.caddy` use `respond`. Only the places that record no reason — the internal-only pattern in `00-acl.caddy`, the catch-all in the Caddyfile — use `abort`.

## Importing with no argument: adapt passes, startup dies (the `path` case)

Here is what happens when a snippet using `{args[:]}` gets no argument.

```sh
$ caddy adapt    --config Caddyfile   # passes. comes out as "not":[{"path":null}]
$ caddy validate --config Caddyfile   # caught here
$ caddy run      --config Caddyfile
Error: ... provision http.matchers.not: loading matcher sets:
       module name 'path': module value cannot be null
```

`adapt` only converts the Caddyfile to JSON; it never loads the modules. `validate` runs through provisioning and catches this. **Use `validate`, not `adapt`, in deployment scripts.**

It is also why `probe-app` and `probe-ext` in this repo make you say `/__none__` **explicitly** when nothing applies. Paths begin with `/`, so `/__none__` matches nothing.

## An argless `remote_ip` neither dies nor matches

The opposite of the `path` case above. With no argument given to a `{args[:]}` snippet, the outcome depends on which matcher it is.

| matcher | adapt output with no argument | startup |
|--|--|--|
| `path {args[:]}` | `{"path":null}` | **dies** |
| `remote_ip {args[:]}` | `{"remote_ip":{}}` | **comes up, and matches nothing** |

Measured on Caddy v2.11.4. Use `import gate-stub` with no argument, attach it to `:8081` and send a request, and the handler behind it answers instead of the stub.

```sh
8081 (no argument)             → PASSTHROUGH   ← the snippet is entirely void
8082 (argument, outside range) → PASSTHROUGH
8083 (argument, inside range)  → stub response
```

Get the `path` case wrong and the proxy refuses to start, so you find out at once. Get the `remote_ip` case wrong and **it keeps running while you believe it is blocking.** When writing a new IP-based exception or blocking snippet, try it once with the argument removed and confirm it behaves as intended.

## Editing the Caddyfile on the host does not reach the container

`conf.d/` is a **directory** mount, so host edits are visible immediately. The `Caddyfile` is a **file** mount, and they are not.

```yaml
- ./Caddyfile:/etc/caddy/Caddyfile:ro   # file — pinned to an inode
- ./conf.d:/etc/caddy/conf.d:ro         # directory — changes inside it follow
```

Docker attaches a file mount to that file's **inode**. Editors generally save by writing a new file and renaming over the old one (`vi`, `sed -i`, `cp`, `mv` all do), which changes the inode on the host, and the container **keeps reading the old one**.

```sh
$ stat -c %i Caddyfile                                   # 255164
$ docker compose exec caddy stat -c %i /etc/caddy/Caddyfile   # 252476
```

So editing the `Caddyfile` and running `caddy reload` **loads the old content again**. No error appears, so you believe it took. The container has to be recreated.

```sh
docker compose up -d --force-recreate caddy
```

Adding and editing sites all happens in `conf.d/`, so `reload` is enough day to day. The `Caddyfile` itself only changes for global options or the catch-all, and only then is a recreate needed.

> ⚠ A directory mount follows changes **inside** it, not a replacement **of** it. Delete `conf.d/` and recreate it — which a deploy script doing `rm -rf conf.d && tar -x` does — and the directory's own inode changes, leaving the container on the old one. Measured: the container then sees an empty directory and `caddy validate` fails with `File to import not found: logsite`, while the running configuration keeps serving from memory, so nothing looks broken until the next restart. Replace the files inside the directory, or recreate the container.

## An empty conf.d comes up as an empty configuration, with no error

```sh
$ caddy adapt --config Caddyfile      # import conf.d/*.caddy matched nothing
{"admin":{"disabled":true}}           # this is not an error
```

The proxy starts normally and does nothing. The cause never reaches the log, so you lose time on "it is up, why does nothing work". That is why `conf.d/public.example.com.caddy.example` ships with the repo — drop the `.example` and there is one site answering immediately, which gives you something to check the stack against right after install.

## `*` in a path matcher also matches the empty string

```
@x path */.env*
   /.env                  BLOCK      ← the leading * is the empty string
   /sub/.env              BLOCK
   /a/b/c/.env.local      BLOCK
```

The rooted form, conversely, misses everything below the root.

```
@b path /.env*
   /.env                  BLOCK
   /sub/.env              allowed    ← missed
```

One `*/x*` covers both the root and everything under it, so there is no need to pair `/x*` with `*/x*`.

## Path matchers ignore case

```
@c path /phpmyadmin*
   /phpMyAdmin/index.php  BLOCK
   /PHPMYADMIN/           BLOCK
```

Variant spellings need no separate entries. Paths are also normalised before matching, so traversal like `/x.jspa/../y.php` cannot slip past. Query strings are not part of the path.

## handle gets reordered, route keeps your order

Caddy reorders `handle` blocks by path specificity. Written order is not evaluation order.

When order is a security boundary — "this path always authenticates, internal or not" — wrap it in a `route` and pin the order by hand.

```caddy
route {
	handle /admin* {         # guaranteed to come first
		route { forward_auth ... }
	}
	handle @internal {
		reverse_proxy ...
	}
	handle {
		route { forward_auth ... }
	}
}
```

For the same reason `log-oidc` has to sit inside a `route`, on the line **after** `forward_auth`. With `handle` alone it is reordered, runs before the headers are attached, and `auth_email` stays empty.

## Why the snippets are wrapped in handle

In Caddy's directive order, `handle` is evaluated before `respond` or `abort`. A blocking snippet built on a bare `respond` is therefore **bypassed entirely on any site that uses `handle` blocks**.

```caddy
# do not write it this way — it is void on sites that use handle
(badbots) {
	@bot header_regexp User-Agent (?i)GPTBot
	respond @bot 403
}
```

Every blocking snippet in this repo is wrapped in `handle`, and has to be **imported before any other handle** in the site block.

## A snippet's named matchers work at site scope

A named matcher defined inside a snippet can be referenced from anywhere in that site's `route` and `handle` blocks once the snippet is imported. That is how `00-acl.caddy` provides `@internal`.

The import does have to be **at the top of the site block**. Import it inside a `route` or `handle` and the matcher is trapped in there.

## {args[:]} works in matcher position

Variadic arguments expand correctly in matcher position.

```caddy
(alien) {
	@a {
		path *.php *.jsp *.asp
		not path {args[:]}
	}
	...
}
```

It is not substituted in the **value** position of a `header` directive, however. That is why `secure-embed` in `00-secure.caddy` uses `{args[0]}` and makes you quote multiple values into a single argument.

## In directive order, abort comes before reverse_proxy

```caddy
site.example {
	import acl-internal
	abort @not_internal       # this runs first
	reverse_proxy http://backend
}
```

Regardless of the order written in the file. Rather than relying on that, `handle @internal { ... } handle { abort }` is clearer to whoever reads it next, and that is how the examples in this repo are written.

## A User-Agent header with an empty value gets through

`not header User-Agent *` catches requests where the header is **absent**. Send the header with an empty string and it passes, because `header <name> *` asks whether the field exists.

```
no UA header at all     → 403
UA: (empty value)       → allowed
```

Real scanners mostly send no UA at all or stamp their own name, so this rarely matters in practice. To catch the empty value too, add `@empty header_regexp User-Agent ^$` separately.

## remote_ip is meaningless without network_mode: host

Bring the stack up on a bridge network with port mapping and every request's `remote_ip` reads as the Docker gateway address. Every IP-based matcher including `acl-internal` stops working, and **no error appears** — everything is simply judged internal, or everything external.

That is the first reason `docker-compose.yml` here uses `network_mode: host`.

## Environment variables work in import paths

```caddy
import sites/{$INSTANCE}/*.caddy
```

This allows several instances to clone one repo and separate only their site definitions. But an empty `INSTANCE` produces a **silently empty configuration**, exactly like the "empty conf.d" entry above, so anyone using it needs their own guard for a missing value. This repo chose simplicity and reads only `conf.d/*.caddy`.
