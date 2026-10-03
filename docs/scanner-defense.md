# Scanner detection

[🇺🇸 English](scanner-defense.md) · [🇰🇷 한국어](scanner-defense.ko.md)

## The claim

> [!IMPORTANT]
> **(first principle)** A request looking for something this site does not have is reconnaissance, not a visit.

A reverse proxy knows what sits behind it. When you know Tomcat is running and a `.php` request arrives, that is not a user who took a wrong turn — it is a scanner reading what is there. This is certainty, not probability: the backend cannot execute PHP, so no legitimate intent can exist behind that request.

That is what separates this from User-Agent bot blocking. A UA is self-reported, so lying costs nothing. What your backend is capable of executing is not something a scanner gets to change.

## Why once is enough

Blocking normally reads "ban after N failures", because failure can be innocent — people mistype passwords.

These requests **cannot be innocent**, so no threshold is needed. One sweep settles it on the spot. In real proxy logs, an address probing for `.php` goes on to hit `/wp-config.php`, `/.ssh/id_rsa`, `/backup.sql` and `/_vti_pvt/service.pwd` within the same session. There is nothing to be gained by waiting for the second one.

## Three layers

One claim, but split into layers because **a different party declares the exception** in each.

| layer | the question | per-site exception |
|--|--|--|
| `probe-secret` | is this a secret or credential file? | **none** |
| `probe-app` | is this the fingerprint of an *app* that is not here? | the real paths, if you are that app |
| `probe-ext` | is this the extension of a *language* that is not here? | your language plus upload namespaces |

### ① probe-secret

`/.env`, `/.ssh/id_rsa`, `/.git/config`, `/.aws/credentials`, `/rclone.conf` and friends. No web application serves these as a feature, so there is no per-site exception.

```caddy
import probe-secret
```

> ⚠ When adding dotfile patterns, make sure `/.well-known/*` does not match. ACME renewal and OIDC discovery both arrive there.

### ② probe-app

`/wp-login.php`, `/wp-content/*`, `/phpmyadmin/`, `/xmlrpc.php` — the fingerprint of **an app that is not here**. Not running WordPress today does not mean never, so this one has to be reversible per site.

```caddy
# not running WordPress
import probe-app /__none__

# running WordPress — on that site these become real paths
import probe-app /wp-admin/* /wp-login.php /wp-content/* /wp-includes/* /xmlrpc.php
```

### ③ probe-ext

Extensions your backend cannot execute. The argument takes two kinds of thing:

**ⓐ the extensions your backend really executes** · **ⓑ the paths where user uploads land**

ⓑ is the part that makes this design work.

## Why the extension alone is not enough

Say an example `.py` file is attached to a wiki page and people download it. The backend is PHP and `.py` never executes. That request is still **legitimate** — it is a download, not an execution request.

An extension cannot tell "execute this" from "send me this file". The **path** can. Uploads only ever land under a namespace the application defines.

```caddy
import probe-ext *.php *.php/* /images/*
```

```
/images/example.py        allowed   ← an example file uploaded to the wiki
/images/nested/a/b.pl     allowed   ← recurses into subdirectories
/shell.py                 444       ← a probe thrown at the root
/admin.jsp                444
```

## Arguments per application

| application | backend | `import probe-ext` arguments |
|--|--|--|
| MediaWiki | PHP | `*.php *.php/* /images/*` |
| DokuWiki | PHP | `*.php *.php/*` |
| WordPress | PHP | `*.php *.php/* /wp-content/uploads/*` |
| LibreNMS · Cacti · RackTables | PHP | `*.php *.php/*` |
| Confluence | Tomcat | `*.jsp *.jspa *.action /download/attachments/* /download/resources/*` |
| Jira | Tomcat | `*.jsp *.jspa /secure/attachment/*` |
| Crowd | Tomcat | `*.jsp` |
| Keycloak | Java | `/__none__` |
| MRTG | CGI | `*.cgi *.cgi/*` |
| NetBox · Django family | Python | `/media/*` |
| Mattermost · Outline · Node/Go generally | — | `/__none__` |

> ⚠ `*.do` and `*.action` are **real** extensions for Java web applications (Struts, Confluence). On a Java backend they must be in the argument list, or the site dies immediately.

> ⚠ **Do not guess** the upload namespace. Run observation mode and the log will tell you (below).

## Requests without an extension are never evaluated

This is what makes the layer safe. Most legitimate web application traffic carries no extension at all.

```
/rest/api/2/issue                      not evaluated
/api/v4/users                          not evaluated
/plugins/servlet/gadgets               not evaluated
/s/abc-CDN/of23ld/9422/batch.js        not evaluated
```

URLs where mod_rewrite has removed the extension behave the same way. Rewriting only ever works **in this design's favour**: strip an extension and the request leaves the evaluated set; allow one and you add it to the arguments.

## Observation mode — start here, always

Nothing is blocked; a `would_block` tag is recorded instead. With banning wired up, one missing upload namespace cuts off real users, so run this for a week or two and confirm zero false positives before switching over.

```caddy
import probe-secret
import probe-app-observe /__none__
import probe-ext-observe *.php *.php/*
```

```sh
# only requests tagged would_block, minus the ones an earlier layer already blocked —
# one request can hit two layers, and without the filter a blocked probe looks like a
# false-positive candidate.
jq -r 'select(.would_block) | select(.block_reason | not)
         | "\(.would_block)\t\(.request.client_ip)\t\(.request.uri)"' \
  logs/*/access.log | sort | uniq -c | sort -rn | head -50
```

**Real traffic in that list is exactly the namespace you missed.** Add it to the arguments and keep observing. Once only scanners remain, drop `-observe` and let it block.

Introducing this to a proxy that is already running lets you do the same check immediately against history: push every unique path in the accumulated access logs through the new matcher and look at whether anything legitimate matches.

## Why blocks use `respond 444`

`abort` drops the connection and **discards every field** added by `log_append`. Measured:

| approach | response | `block_reason` |
|--|--|--|
| `log_append` + `abort` | connection dropped | **lost** |
| `log_append` + `respond 403` | 403 | recorded |
| `log_append` + `error 403` | 403 | recorded |
| `log_append` + `respond 444` | 444 (empty body) | recorded |

Without a reason in the log, fail2ban has to reproduce the path list as a regex, and from that moment the two drift apart. `444` is a non-standard code with no body, so what a scanner learns is effectively the same as from `abort`. The evidence is in [Caddy gotchas](caddy-gotchas.md).

## Why each layer takes its own arguments

Merge the three into one argument list and the thing opens up. Measured:

A MediaWiki site declares `*.php` as "my language". Share the arguments and that declaration also applies to `probe-app`, where `/wp-login.php` matches `*.php` and is therefore **exempted**. The WordPress fingerprint layer stops working entirely.

```
shared arguments:    MediaWiki site → /wp-login.php → allowed, 200   ✗
separate layers:     MediaWiki site → /wp-login.php → 444            ✓
```

## Wiring up fail2ban

This repo covers **detection and logging**. Banning is separate.

```
block_reason = probe-secret | probe-app | probe-ext     ← one-strike material
             = ai-training-crawler | empty-user-agent   ← UA-based, depends on policy
             = ai-assistant-fetch
```

Always put your internal ranges in `ignoreip`. Run a vulnerability scan from inside and you will ban yourself. See [Logging](logging.md) for the schema.
