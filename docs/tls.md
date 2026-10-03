# TLS

[🇺🇸 English](tls.md) · [🇰🇷 한국어](tls.ko.md)

Caddy obtains certificates on its own. Put a host name in a site block and it issues from Let's Encrypt over HTTP-01 by default. Configuration is only needed when that cannot work.

## Which method

| situation | method | configuration |
|--|--|--|
| the domain points here and port 80 is open to the internet | HTTP-01 | **none** (the default) |
| port 80 is unreachable from the internet | DNS-01 | `import dns01` plus a plugin build |
| no domain yet, or internal testing | self-signed | `tls internal` |

## HTTP-01 (default)

This is what you get by doing nothing.

```caddy
app.example.com {
	import logsite app.example.com
	reverse_proxy http://127.0.0.1:8080
}
```

Two conditions. The A/AAAA record for `app.example.com` has to point at this server, and Let's Encrypt has to be able to **reach port 80 directly**. A firewall in front of 80 means failure.

Fill in `ACME_EMAIL` in `.env` to get expiry warnings by mail.

## Self-signed

For when the domain is not ready, or the host is internal only.

```caddy
app.example.com {
	tls internal
	...
}
```

Browsers warn. `curl` needs `-k`. Moving to a public certificate later is a matter of deleting the `tls internal` line.

The bundled `conf.d/public.example.com.caddy.example` works this way. `example.com` is reserved for documentation (RFC 2606), so it cannot point at your server and ACME issuance is impossible.

## DNS-01

Use this when port 80 is unreachable from the internet — an internal-only host, or anything behind a firewall. Validation happens through a DNS record, so no inbound connection is needed.

This repo ships configuration for RFC2136 (dynamic DNS update).

### ① Build an image with the plugin

Stock `caddy:2` carries no DNS plugins.

```sh
vi docker-compose.yml
```

```yaml
    # image: caddy:2
    build: ./caddy
```

```sh
docker compose up -d --build
```

For another provider (Cloudflare, Route53 and so on), change `--with` in `caddy/Dockerfile`; the list is at <https://github.com/caddy-dns>. You then also have to rewrite the `dns rfc2136 { ... }` block in `conf.d/00-tls-dns01.caddy` in that provider's syntax.

### ② Delegate DNS

`_acme-challenge.<hostname>` has to be NS-delegated to the name server that accepts your TSIG updates.

### ③ TSIG key

```sh
vi .env
```

```
DNS_TSIG_KEYNAME=acme-update
DNS_TSIG_ALG=hmac-sha256
DNS_TSIG_SECRET=...
DNS_TSIG_SERVER=ns.example.com:53
```

### ④ Use it on a site

```caddy
app.example.com {
	import dns01
	import logsite app.example.com
	reverse_proxy http://127.0.0.1:8080
}
```

> ⚠ Miss any one of those three and issuance fails, and Caddy falls back to self-signed. You get a browser warning with no idea why unless you read the log.

```sh
docker compose logs caddy | grep -i acme
```

## Where certificates live

In the `caddy_data` named volume, together with the **ACME account key**.

Do not turn it into a bind mount. Delete it by accident and the account key goes with it, and Let's Encrypt has issuance limits.

```sh
docker compose down       # volume kept. safe
docker compose down -v    # volume deleted. the account key goes too
```

## Wildcards

DNS-01 only. HTTP-01 cannot issue them.

```caddy
*.example.com {
	import dns01
	import logsite wildcard.example.com
	...
}
```

A wildcard site block is matched **after** any explicitly named host. If an `app.example.com` block exists, it wins.

## When it does not work

**"could not get certificate"** — check that DNS points here and that port 80 is open from outside.

```sh
dig +short app.example.com
curl -I http://app.example.com/.well-known/acme-challenge/test
```

**`/.well-known/` is being blocked** — if you edited a blocking snippet, make sure `/.well-known/*` does not match it. Certificate renewal and OIDC discovery both arrive there. The shipped configuration does not catch it.

**Rate limits** — Let's Encrypt caps issuance per domain per week. Test against staging before burning that budget on repeated failures.

```caddy
{
	acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
}
```

Staging certificates are not trusted by browsers; they only confirm the configuration is right. Afterwards, delete the line, clear the staging account with `docker compose down -v`, and bring the stack back up.
