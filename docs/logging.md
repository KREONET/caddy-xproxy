# Logging

[🇺🇸 English](logging.md) · [🇰🇷 한국어](logging.ko.md)

## Where

```
logs/<hostname>/access.log        per site
logs/_catchall/access.log         direct IP access · unknown hosts · SNI misses
```

Rolled at 50 MiB, 14 files kept for 90 days. Rolled files land as `access-<timestamp>-size.log.gz`.

Splitting per site keeps a scanner flood on one host from pushing another host's log out of the window.

## Why _catchall matters

Anything that matched no site in `conf.d/` ends up here: requests that arrived by IP, unknown host names, connections whose SNI did not line up.

**A real user has no reason to be here.** That makes this file the cleanest answer to "who is sweeping our address range".

## Schema

Caddy's default JSON log, with this repo's fields appended.

```json
{
  "ts": 1789174679.2,
  "request": {
    "client_ip": "203.0.113.9",
    "host": "app.example.com",
    "method": "GET",
    "uri": "/wp-login.php",
    "headers": {"User-Agent": ["curl/8.18.0"]}
  },
  "status": 444,
  "size": 0,
  "duration": 0.0008,
  "block_reason": "probe-app"
}
```

Use `client_ip`. There is a `remote_ip` too, but it differs once a proxy chain is involved.

## Appended fields

| field | values | written by |
|--|--|--|
| `block_reason` | `probe-secret` · `probe-app` · `probe-ext` | the three scanner layers |
| | `ai-training-crawler` · `empty-user-agent` · `ai-assistant-fetch` | badbots |
| | `geo-denied` | geo-allow |
| `would_block` | `probe-app` · `probe-ext` | observation mode (nothing is blocked). Also tags requests an earlier layer already blocked |
| `auth` | `oidc` · `internal-bypass` | sites behind the OIDC gate |
| | `gate-stub` | a request terminated because it cannot finish the OIDC flow |
| `auth_sub` | the OIDC sub claim | when `auth=oidc` |
| `auth_email` | email address | when `auth=oidc` |

## Reading status

| status | meaning |
|--|--|
| `444` | blocked by a scanner layer or by `geo-allow`. Empty body |
| `403` | blocked by badbots |
| `0` | connection dropped — `abort`. External access to an internal-only site, or the catch-all |
| anything else | the backend answered |

`status: 0` carries **no** `block_reason`, because `abort` discards the `log_append` fields. That is why every block that has to record a reason uses `respond`. The evidence is in [Caddy gotchas](caddy-gotchas.md).

## Queries worth keeping

**Blocks by reason**

```sh
jq -r 'select(.block_reason) | .block_reason' logs/*/access.log \
  | sort | uniq -c | sort -rn
```

**The loudest scanner addresses**

```sh
jq -r 'select(.block_reason) | .request.client_ip' logs/*/access.log \
  | sort | uniq -c | sort -rn | head -20
```

**What observation mode caught — real traffic in here is a namespace you forgot**

```sh
jq -r 'select(.would_block) | select(.block_reason | not)
         | "\(.would_block)\t\(.request.uri)"' logs/*/access.log \
  | sort | uniq -c | sort -rn | head -50
```

**What one person did (sites behind the gate)**

```sh
jq -r 'select(.auth_email=="alice@example.com")
       | "\(.ts)\t\(.status)\t\(.request.uri)"' logs/app.example.com/access.log
```

`auth_email` can change. For tracking over time key on `auth_sub` instead — the OIDC sub claim does not change.

**Including rolled logs**

```sh
zcat -f logs/*/access*.log* | jq -r 'select(.block_reason) | .request.client_ip' \
  | sort | uniq -c | sort -rn | head
```

## Wiring up fail2ban

This repo handles detection and logging; banning is somebody else's job. Everything needed is in the log.

```ini
# roughly this shape
[Definition]
failregex = ^.*"client_ip":"<HOST>".*"block_reason":"probe-(secret|app|ext)".*$
```

A few things to watch.

**Do not match on `status`.** A request dropped by `abort` is `status: 0` with no reason attached. `block_reason` is the only key you can trust.

**Put your internal ranges in `ignoreip`.** Run a vulnerability scan from inside and you will ban yourself.

**Account for log rolling.** Caddy rolls by renaming and opening a new file. fail2ban follows the inode change, but keep the `logpath` glob matching only `access.log`. Catch the rolled `.gz` files too and old entries re-ban people.

**Run observation mode before going to one strike.** `probe-*` should have no false positives by design, but miss one upload namespace and you cut off real users. See [Scanner detection](scanner-defense.md) for the rollout.

## When no logs appear

Check the permissions on the host's `logs/` directory; Caddy inside the container has to be able to write there. Per-site subdirectories appear on the first request — add a site that nobody has visited yet and there is no directory either.
