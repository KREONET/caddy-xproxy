# Install

[🇺🇸 English](install.md) · [🇰🇷 한국어](install.ko.md)

## Requirements

* Docker and Docker Compose
* **Ports 80 and 443 free.** The stack runs with `network_mode: host`, so it cannot share a host with an existing web server. Clear Apache or nginx out of the way first.
* For public certificates, a domain pointing at this server with port 80 reachable from the internet ([TLS](tls.md))

`network_mode: host` is there for two reasons: reaching backends on the internal network directly, and letting `remote_ip` matchers see the real client address. Switch to bridge with port mapping and every IP-based ACL stops working without saying so.

## First boot

```sh
git clone https://github.com/kreonet/caddy-xproxy /opt/xproxy
cd /opt/xproxy
cp .env.example .env
cp conf.d/public.example.com.caddy.example conf.d/public.example.com.caddy
docker compose up -d
```

Every site file in `conf.d/` ships with an `.example` suffix. The glob is `*.caddy`, so nothing loads until the suffix comes off. Enabling one sample as above gives you something to check against right after install.

`.env` can stay as it is for now. Filling in `ACME_EMAIL` is what gets you expiry warnings once real certificates are issued.

Check it:

```sh
curl -k --resolve public.example.com:443:127.0.0.1 https://public.example.com/
# caddy-xproxy is running. Replace this block with your own site.
```

`example.com` is reserved for documentation (RFC 2606) and cannot point at your server, so Caddy issues a self-signed certificate instead. That is why `-k` is needed.

## It started but nothing answers

With no `*.caddy` file in `conf.d/`, Caddy comes up **without an error** on a configuration that does nothing. The bundled examples are all `.example`, so this is the state until you drop a suffix.

```sh
ls conf.d/*.caddy      # at least one site file has to exist
```

## Changing configuration

No restart needed. `conf.d/` is a **directory** bind mount, so edits on the host are visible immediately.

> ⚠ The `Caddyfile` itself is different. It is a **file** mount, pinned to an inode, so the container keeps reading the old content no matter what you do on the host, and `reload` quietly loads the previous configuration. After changing global options or the catch-all, `docker compose up -d --force-recreate caddy` is required. See [Caddy gotchas](caddy-gotchas.md).

```sh
# check syntax first — a broken reload keeps the old config, but check anyway
docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile

docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

> ⚠ Check with **`caddy validate`**, not `caddy adapt`. `adapt` only looks at syntax and lets real errors through — a missing `{args[:]}` argument being the usual one, which passes `adapt` and then kills startup. `validate` runs through provisioning and catches it. See [Caddy gotchas](caddy-gotchas.md).

A failed reload leaves the running configuration untouched. Read the log.

```sh
docker compose logs --tail 50 caddy
```

## Bringing up the OIDC gate

```sh
cp oauth2-proxy/oidc.cfg.example oauth2-proxy/oidc.cfg
vi oauth2-proxy/oidc.cfg          # issuer, client_id, redirect_url
vi .env                            # O2P_CLIENT_SECRET, O2P_COOKIE_SECRET
docker compose --profile oidc up -d
```

Without `--profile oidc`, a plain `up -d` leaves oauth2-proxy down. See [OIDC gate](oidc.md).

## When DNS-01 is needed

This is for when port 80 is unreachable from the internet and HTTP-01 is out. You have to build an image with the rfc2136 plugin.

```sh
vi docker-compose.yml     # comment out image: caddy:2 and enable build: ./caddy
vi .env                   # fill in DNS_TSIG_*
docker compose up -d --build
```

See [TLS](tls.md).

## Updating

```sh
cd /opt/xproxy
git pull
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

Your own site files in `conf.d/`, `.env` and `oauth2-proxy/*.cfg` are either ignored or new, so `git pull` does not overwrite them. Editing the `00-*.caddy` snippets does cause conflicts — leave the snippets alone and tune through arguments in your site file instead.

To move the image itself forward:

```sh
docker compose pull && docker compose up -d
```

## Where the logs are

```
logs/<hostname>/access.log        per site
logs/_catchall/access.log         direct IP access, unknown hosts, SNI misses
```

See [Logging](logging.md) for the schema.

## Removing

```sh
cd /opt/xproxy
docker compose down
```

Issued certificates and the ACME account key live in a named volume, so `down` alone does not remove them. `docker compose down -v` clears everything, but **it takes the ACME account key with it.** Keep the volume if you plan to come back — Let's Encrypt has issuance limits.
