# Adding a site

[🇺🇸 English](sites.md) · [🇰🇷 한국어](sites.ko.md)

## The basic rule

One file in `conf.d/` is one site. The name is yours to pick — `import conf.d/*.caddy` reads them all — but matching the host name makes files easy to find.

```sh
cp conf.d/public.example.com.caddy.example conf.d/app.example.com.caddy
vi conf.d/app.example.com.caddy
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

**Files ending in `.example` are not loaded.** The glob is `*.caddy`, which never matches `*.caddy.example`. Drop the suffix to enable an example; conversely, appending something like `.disabled` takes a site down for a while.

The two bundled examples are written to **contrast** with each other. Put them side by side and it becomes clear why each line differs.

| file | what it shows |
|--|--|
| `public.example.com.caddy.example` | public site, PHP backend (MediaWiki), upload namespace exempted, search engines and AI summaries allowed |
| `inside.example.com.caddy.example` | internal only, Node backend (Outline), authentik OIDC gate, internal ranges pass unauthenticated |

| | `public` | `inside` |
|--|--|--|
| who gets in | anyone | internal ranges plus anyone logged in |
| `robots` | `robots-allow` | `robots-deny` |
| `ai-assist-deny` | **not used** (a public wiki people read) | used |
| `probe-ext` | `*.php *.php/* /images/*` | `/__none__` |
| authentication | none | `acl-internal` plus `forward_auth` |

## A minimal site

```caddy
app.example.com {
	import logsite app.example.com
	import secure
	import badbots

	import probe-secret
	import probe-app /__none__
	import probe-ext /__none__

	reverse_proxy http://127.0.0.1:8080
}
```

If the host name resolves to this server in public DNS, the certificate appears on its own. With no DNS yet, add a `tls internal` line to come up self-signed and delete that line later.

## Import order

**Import the blocking snippets before any other `handle`.** Caddy reorders `handle` blocks by path specificity, so putting them later lets the site's catch-all `handle` match first and the blocking is bypassed entirely.

The recommended order:

```caddy
app.example.com {
	import logsite app.example.com      # 1. logging
	import secure                        # 2. headers
	import robots-deny                   # 3. robots.txt
	import badbots                       # 4. UA blocking
	import probe-secret                  # 5. the three scanner layers
	import probe-app /__none__
	import probe-ext /__none__
	import acl-internal                  # 6. matcher definitions, if any

	handle ... {                         # 7. routing comes after
	}
}
```

## Routing by path

```caddy
wiki.example.com {
	import logsite wiki.example.com
	import secure
	import badbots
	import probe-secret
	import probe-app /__none__
	import probe-ext *.php *.php/* /images/*

	handle /api/* {
		reverse_proxy http://127.0.0.1:9000
	}
	handle /static/* {
		root * /srv/static
		file_server
	}
	handle {
		reverse_proxy http://127.0.0.1:8080
	}
}
```

`handle` blocks are mutually exclusive: the first match ends it. The final `handle` with no argument is the catch-all.

**When order is a security boundary** — "this path always authenticates, internal or not", say — wrap it in a `route` to pin the order. Inside a `route`, written order is evaluation order.

```caddy
route {
	handle /admin* {
		route { forward_auth ... }      # has to come first
	}
	handle @internal {
		reverse_proxy ...
	}
	handle {
		route { forward_auth ... }
	}
}
```

## When the route tree is needed twice

Internal unauthenticated, external logged in — with more than one backend, that shape makes you write the same `handle` group twice. Copy it and the day comes when only one copy gets fixed. **Define a snippet inside that site's file.**

```caddy
# route tree used only by this site.
(app_routes) {
	handle /api/* {
		reverse_proxy http://127.0.0.1:9000
	}
	handle {
		reverse_proxy http://127.0.0.1:8080
	}
}

app.example.com {
	import logsite app.example.com
	...
	import acl-internal

	handle @internal {
		import log-internal
		import app_routes
	}
	handle {
		route {
			forward_auth 127.0.0.1:4180 { ... }
			import log-oidc
			import app_routes
		}
	}
}
```

A snippet definition can live anywhere and needs no `00-` prefix — `(app_routes)` only has to appear above the site block. Since it belongs to that site rather than everyone, keep it in the site file instead of promoting it to `conf.d/00-*.caddy`.

## Choosing probe-ext arguments

Two things go in: the backend's language, and the upload paths. The per-application table is in [Scanner detection](scanner-defense.md).

**If you do not know the upload path, do not guess — run observation mode first.**

```caddy
import probe-ext-observe *.php *.php/*
```

A week or two later the log shows what you missed. Filter out requests that already carry a `block_reason`: an earlier layer's block is tagged `would_block` as well, so without the filter something like `/wp-login.php` reads as "a legitimate path probe-ext nearly caught".

```sh
jq -r 'select(.would_block) | select(.block_reason | not)
         | "\(.would_block)\t\(.request.uri)"' logs/*/access.log \
  | sort | uniq -c | sort -rn | head -50
```

## Internal ranges

The default in `conf.d/00-acl.caddy` is the RFC1918 private ranges. Edit **that one file** for your environment.

```caddy
(acl-internal) {
	@internal remote_ip 10.0.0.0/8 192.0.2.0/24 198.51.100.0/24
}
```

Copy the range list into each site and it will drift — one site ends up with the office WiFi range and another does not.

**Per-site exceptions** (a partner range, a health-check address) do not go in the shared table. Attach them in that site's file as a separate matcher, so that "why does this one IP get through" stays next to the answer.

```caddy
import acl-internal

# Health-check monitor. Cannot do browser auth, so it is exempt.
@healthcheck remote_ip 198.51.100.7/32

handle @internal {
	import log-internal
	reverse_proxy http://127.0.0.1:8080
}
handle @healthcheck {
	import log-internal
	reverse_proxy http://127.0.0.1:8080
}
handle {
	route { forward_auth ... }
}
```

## Common mistakes

**Using `import probe-ext` with no argument** — `caddy validate` passes and **the proxy dies on reload**. Where nothing applies, say `/__none__`.

**Leaving `*.action` out on a Java backend** — Confluence dies immediately. `.do` and `.action` are real extensions for Java web applications.

**Emptying `conf.d/`** — you get a proxy that starts without error and does nothing.

**Editing the `00-*.caddy` snippets** — conflicts on `git pull`. Tune through arguments in the site file instead. The range list in `00-acl.caddy` is the exception; it is meant to be edited.

**Adding `ai-assist-deny` to a public wiki** — asking an AI tool to summarise a link then returns 403. Leave it off anything meant for people to read.
