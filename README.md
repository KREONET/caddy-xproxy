# caddy-xproxy — a batteries-included Caddy reverse-proxy stack

[🇺🇸 English](README.md) · [🇰🇷 한국어](README.ko.md)

A reverse proxy that ships with ready-made snippets — scanner and bot blocking, an OIDC gate, and more.

## Install

```sh
git clone https://github.com/kreonet/caddy-xproxy /opt/xproxy
cd /opt/xproxy
cp .env.example .env
cp conf.d/public.example.com.caddy.example conf.d/public.example.com.caddy
docker compose up -d
```

Drop the `.example` suffix from the bundled sample site and a fresh clone already answers.

```sh
curl -k --resolve public.example.com:443:127.0.0.1 https://public.example.com/
# caddy-xproxy is running. Replace this block with your own site.
```

Details in [Install](docs/install.md).

## Requirements

* Docker and Docker Compose
* Ports 80 and 443 free — the stack runs with `network_mode: host`
* For public certificates, a domain pointing at this server with port 80 reachable ([TLS](docs/tls.md))

## Adding a site

One file in `conf.d/` is one site. Copying the bundled sample is the fastest way in.

```sh
cp conf.d/public.example.com.caddy.example conf.d/app.example.com.caddy
vi conf.d/app.example.com.caddy
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

```caddy
app.example.com {
	import logsite app.example.com
	import secure
	import badbots

	import probe-secret
	import probe-app /__none__
	import probe-ext *.php *.php/* /uploads/*

	reverse_proxy http://127.0.0.1:8080
}
```

Two examples ship with the repo, written to contrast with each other: `public.example.com.caddy.example` (public, PHP backend, upload namespace) and `inside.example.com.caddy.example` (internal-only, Node backend, OIDC gate). Files ending in `.example` are not loaded; drop the suffix to enable them. See [Adding a site](docs/sites.md).

## Ready-made snippets

| import | What it does |
|--|--|
| `logsite <host>` | Per-site JSON access log with rolling |
| `secure` · `secure-embed` | Attaches HSTS, CSP and friends; hides the server banner |
| `probe-secret` · `probe-app` · `probe-ext` | Three-layer scanner detection |
| `badbots` · `ai-assist-deny` · `badbots-ua-ok` | Crawler blocking by User-Agent |
| `robots-allow` · `robots-deny` | robots.txt answered by Caddy itself |
| `acl-internal` | Internal network matcher, defined in exactly one place |
| `geo-allow` | Drop traffic from outside a country (list is generated) |
| `gate-stub` | Terminate clients that cannot complete the OIDC flow |
| `log-oidc` · `log-internal` | Record *who* in the access log |
| `dns01` | DNS-01 certificate issuance |

Per-snippet arguments and caveats are in [Snippet reference](docs/snippets.md).

## Scanner detection

One claim drives it: **a request looking for something this site does not have is reconnaissance, not a visit.**

A `.php` request against a Tomcat backend. A `/wp-login.php` on a host that does not run WordPress. A `/.env` that no application ever serves. Each of those is sufficient evidence on the first try. Every site declares what it actually has, in one line; everything else is treated as a probe.

Requests without an extension — `/api/v4/users`, SPA routes, rewritten URLs — are never evaluated at all. Content that happens to carry a foreign extension, like an example `.py` file uploaded to a wiki, is exempted by declaring its upload path rather than its extension.

The reason for each block lands in the access log as `block_reason`, ready for fail2ban. When rolling this out, start with observation mode, which tags without blocking. See [Scanner detection design](docs/scanner-defense.md).

## OIDC gate

For backends with no login of their own, or where you want the proxy to stop unauthenticated traffic before it ever reaches the application. Internal networks pass without authentication; everyone else is sent to the IdP. The backend never has to know it is being protected.

```sh
cp oauth2-proxy/oidc.cfg.example oauth2-proxy/oidc.cfg
cp conf.d/inside.example.com.caddy.example conf.d/inside.example.com.caddy
docker compose --profile oidc up -d
```

See [OIDC gate](docs/oidc.md).

## Documentation

Every document has a Korean counterpart at `*.ko.md`.

| | |
|--|--|
| [Install](docs/install.md) | Requirements, first boot, reload, troubleshooting |
| [Adding a site](docs/sites.md) | conf.d rules, routing, common mistakes |
| [Snippet reference](docs/snippets.md) | Arguments and caveats for every import |
| [Scanner detection](docs/scanner-defense.md) | The three layers, per-app table, observation mode |
| [OIDC gate](docs/oidc.md) | oauth2-proxy, forward_auth, internal bypass |
| [TLS](docs/tls.md) | HTTP-01, DNS-01, self-signed |
| [Logging](docs/logging.md) | JSON schema, block_reason, fail2ban |
| [Caddy gotchas](docs/caddy-gotchas.md) | Behaviours confirmed by measurement |

## License

MIT
