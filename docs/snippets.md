# Snippet reference

[🇺🇸 English](snippets.md) · [🇰🇷 한국어](snippets.ko.md)

Every import provided by `conf.d/00-*.caddy`. The `00-` prefix forces load order — a snippet has to be read before the sites that use it.

## logsite

```caddy
import logsite <hostname>
```

Per-site JSON access log. Lands in `logs/<hostname>/access.log`, rolled at 50 MiB, 14 files kept for 90 days.

Splitting per site keeps a scanner flood on one host from pushing another host's log out of the window. See [Logging](logging.md) for the schema.

## secure · secure-embed

```caddy
import secure
import secure-embed https://embed.example.com
import secure-embed "https://a.example https://b.example"
```

Attaches CSP, HSTS, `X-Content-Type-Options` and `X-XSS-Protection`, and removes the `Server` header. `secure-embed` is the same set with `frame-ancestors` widened, for when another site has to embed yours in an iframe.

> ⚠ With more than one origin, pass them **quoted as a single argument**. In the value position of a `header` directive `{args[:]}` is not substituted, which is why this uses `{args[0]}`.

## probe-secret · probe-app · probe-ext

```caddy
import probe-secret
import probe-app /__none__
import probe-ext *.php *.php/* /images/*
```

The three scanner layers. Observation variants exist too.

```caddy
import probe-app-observe /__none__
import probe-ext-observe *.php *.php/*
```

The reasoning, the per-application argument table and the rollout all live in [Scanner detection](scanner-defense.md). Only the caveats are here.

> ⚠ Omit the argument entirely and `caddy validate` passes while **startup dies**. Where nothing applies, say `/__none__` explicitly.

> ⚠ Never merge the three argument lists. A `*.php` declaration then also exempts `/wp-login.php` and the app layer is open.

> ⚠ On a Java backend, `*.action` and `*.do` must be in the arguments.

> ⚠ Import before any other `handle`.

Blocks use `respond 444`. `abort` discards `log_append`, so the reason never reaches the log.

## badbots · ai-assist-deny

```caddy
import badbots
import ai-assist-deny
```

User-Agent based blocking. `badbots` answers 403 to training-data crawlers, bulk scrapers, and requests with no UA at all. `ai-assist-deny` covers the fetch that happens when a person hands an AI tool a link.

> ⚠ **Do not add `ai-assist-deny` to anything meant for people to read**, such as a public wiki. Asking for a summary of a link then returns 403, and whether it works depends on which AI the person uses.

A UA is self-reported, so lying costs nothing. Scanners that do not report are `probe-*`'s job. Neither replaces the other.

`block_reason` values: `ai-training-crawler`, `empty-user-agent`, `ai-assistant-fetch`.

### badbots-ua-ok

```caddy
import badbots-ua-ok
```

`badbots` with **only the empty-UA rule removed**. Crawler blocking is unchanged.

Use it only on sites holding data published for machines to read — XML or JSON feeds, files an external validator fetches. PHP's `file_get_contents()` and `DOMDocument::load()` send **no User-Agent header at all**. Such a fetcher hits the empty-UA matcher in `badbots` and gets 403; upstream it is reported as "empty or not accessible", and all your log shows is `empty-user-agent`, so neither side can see the cause.

## robots-allow · robots-deny

```caddy
import robots-allow
import robots-deny
```

Caddy answers `robots.txt` itself. The intended content goes out even when the backend is down, or when an SPA returns HTML where a 404 belongs.

This is a separate policy from `badbots`. robots.txt is advisory, so it only means anything to crawlers that honour it — search engines. The ones that do not are `badbots`'s job.

## acl-internal

```caddy
import acl-internal
```

**Only defines** the `@internal` matcher. What to do with it is the site's call.

```caddy
# internal passes, external logs in
handle @internal { import log-internal
                   reverse_proxy ... }
handle           { route { forward_auth ... } }

# internal passes, external gets dropped
handle @internal { import log-internal
                   reverse_proxy ... }
handle           { abort }
```

The range list lives in `conf.d/00-acl.caddy` and nowhere else. The default is RFC1918, so it has to be edited for your environment.

> ⚠ Import at the **top** of the site block. Import it inside a `route` or `handle` and the matcher is trapped in there.

> ⚠ For `remote_ip` to see the real client address, compose has to be on `network_mode: host`. Switch to bridge and this ACL silently stops working.

## log-oidc · log-internal

```caddy
route {
	forward_auth 127.0.0.1:4180 { ... }
	import log-oidc
	reverse_proxy ...
}
```

Records *who* in the access log. `log-oidc` writes `auth=oidc` along with `auth_sub` (the OIDC sub claim, a stable identifier) and `auth_email`. `log-internal` writes `auth=internal-bypass`, stating that having no identifier is correct here.

> ⚠ It must sit **inside a `route`, on the line after `forward_auth`**. The values only exist once `forward_auth` has attached the headers through `copy_headers`. With `handle` alone, Caddy reorders and you get empty values.

> Unauthenticated requests carry none of these fields. The handler chain ends at the 401 that redirects to login. This is not a misconfiguration.

## gate-stub

```caddy
import gate-stub 198.51.100.0/24
```

Terminates, **in front of the gate**, clients that cannot complete the OIDC flow. The usual case is a mail security gateway's URL checker: it opens the links inside notification mail sent by an app behind the gate, to judge whether they are phishing. It is not a browser, cannot carry cookies or fill forms, and so never finishes logging in. Why that traffic reaches you at all is covered in [OIDC gate](oidc.md#notification-mail-links-knocking-on-the-proxy).

Let it through and every link walks `/oauth2/start` to the IdP login page (200) and is dropped there, while a callback with no state leaves a 400 behind. The IdP takes pointless load and real authentication failures get buried.

The response is one HTML 200 telling the client to log in. A backend's "not permitted" is usually a 302 to a login page, and what the checker ends up receiving is that page's 200 — so this replaces only that endpoint, in one hop. Imitating the 302 would mean hard-coding each application's login path (`/login.jsp` for Jira, `/login.action` for Confluence) and then exempting that path too, or you get a redirect loop.

> ⚠ Import before any other `handle`. It has to catch the `/oauth2/*` callback as well.

> ⚠ The argument is mandatory. **Omitting it raises an error nowhere.** The `remote_ip` matcher becomes an empty object, matches nothing, and the snippet is silently void. Believing you are blocking is the worst state of all. See [Caddy gotchas](caddy-gotchas.md).

Since this is a clean termination rather than a block, it records `auth=gate-stub` instead of a `block_reason`.

## geo-allow

```caddy
import geo-allow
```

Drops traffic from outside one country, plus whatever ranges you always allow, with `444`. What ships is `conf.d/00-geo.caddy.example`; the list itself is generated.

```sh
./tools/geo-gen.sh kr 203.0.113.0/24 198.51.100.0/24 > conf.d/00-geo.caddy
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

The first argument is the country code; the rest are ranges outside it that must still get through — overseas PoPs, partner networks, monitoring sources.

> ⚠ **Do not put this on a public site.** Travellers, roaming users and overseas mirrors all get cut off.

> ⚠ The list is a snapshot taken when you generated it. Allocations keep moving, so regenerate it periodically.

This is a different layer from `robots-deny`. robots is advisory; this denies access.

## dns01

```caddy
import dns01
```

Issues certificates over DNS-01, for when port 80 is unreachable from the internet.

> ⚠ Requires an image containing the rfc2136 plugin. Stock `caddy:2` does not have one. See [TLS](tls.md) for the build.

## When a snippet has to change

Editing `00-*.caddy` directly means a conflict on `git pull`. Tune through arguments in your site file where you can.

`00-acl.caddy` is the exception: the range list is meant to be edited for your environment.

When a snippet genuinely has to change, adding another file under a new name is safer — `conf.d/01-my-snippets.caddy`, for instance. It loads after `00-`, and it never conflicts with `git pull`.
