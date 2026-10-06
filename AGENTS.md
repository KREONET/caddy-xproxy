# caddy-xproxy — working conventions

[🇺🇸 English](AGENTS.md) · [🇰🇷 한국어](AGENTS.ko.md)

A batteries-included Caddy reverse-proxy stack. What it does is in [README.md](README.md) and [docs/](docs/); this file is only about how to work on it.

Korean-specific writing rules live in [AGENTS.ko.md](AGENTS.ko.md). Read that one too when you touch the Korean documents.

## Deployment overrides

A deployment that clones this repo may keep an `AGENTS.ops.md` of its own. **Where that file differs from this one, it wins.** It describes an operating environment — which sites run, in which language the team works, what the on-call person needs at three in the morning — and this repo cannot know any of that.

Read it first when it exists. Everything it does not mention still follows this file. Keep the deviations in it, not here: this file stays the same for everyone who clones.

## Commits

`git log --oneline` alone should show what came in. Follow Conventional Commits.

```text
<type>(<scope>): <what and why>
```

- **Subject only, no body.** The reasoning belongs in `docs/`. A log full of explanation is knowledge pooling where nobody reads it.
- Subjects are **English, lowercase** after the colon, even though the documents are bilingual.
- `scope` is **optional**. Add it when the subject alone does not say which part moved.

| type | what it covers |
|--|--|
| `feat` | a new snippet or capability |
| `fix` | something that behaved wrongly |
| `docs` | `docs/`, `README*.md`, `AGENTS*.md`, config comments |
| `refactor` | structure or placement, behaviour unchanged |
| `chore` | license, `.gitignore`, build surroundings |
| `release` | release preparation and tagging |

| scope | what it covers |
|--|--|
| `snippets` | `conf.d/00-*.caddy` |
| `samples` | `conf.d/*.caddy.example` |
| `compose` | `docker-compose.yml`, `caddy/Dockerfile` |
| `caddyfile` | `Caddyfile` |
| `tools` | `tools/` |
| document name | for `docs` commits: `readme`, `gotchas`, `oidc`, … |

```text
feat(snippets): add badbots-ua-ok, gate-stub and geo-allow
fix(compose): stop the oidc profile's secret check from blocking up -d
refactor(samples): ship both sample sites as .example
docs(gotchas): warn that editing the Caddyfile misses the container
```

Four rules hold:

- **One commit, one intent.** A session is not a commit. A rename, a behaviour fix and a docs rewrite are three commits even when they touch the same files.
- Write **what and why**, not "changed a file".
- Never name a path in a subject that does not exist at that commit.
- Only `Co-Authored-By: Claude <noreply@anthropic.com>` is trailed. This repo is public, so no `Claude-Session`.

## Documentation

- **Every document is a pair.** `X.md` is English and `X.ko.md` is Korean. Both carry the same switcher line directly under the title, with both languages linked:

  ```markdown
  [🇺🇸 English](X.md) · [🇰🇷 한국어](X.ko.md)
  ```
 Update both or neither. English is the entry point; Korean carries the same content, not a summary.
- **`AGENTS.ko.md` is the one exception, and it is not a translation.** It holds the rules that only apply to writing Korean — register, the phrasings that keep creeping back, which terms stay untranslated. Everything general lives here and is not repeated there. Do not try to bring the two files into line; they are not meant to match.
- **The README stays short.** Install, and the short version of each topic. Everything long lives in `docs/` and the README links to it.
- **No hard wrapping.** Write each paragraph and list item as one long line and let the renderer wrap. Hard wraps make diffs useless — one added word reflows every following line.
- **Do not copy into the docs what the config already says.** Backend addresses, ports, which site uses which gate: all of that is in `conf.d/`. Copy it into a document and only one copy gets fixed, and later you believe the wrong one. Documents carry what has nowhere to live in the config — design reasoning, measurements, migration state that spans files.
- **Numbers are measured, never estimated.** Block counts, path counts, observed behaviour are all real measurements. State the date beside them and re-measure rather than carrying stale figures forward.
- Put a claim that must not be missed in a GitHub alert block.

## Comments in config files

Every configuration file in this repo — snippets, sample sites, `Caddyfile`, `docker-compose.yml`, `.env.example`, `caddy/Dockerfile`, `oauth2-proxy/*.cfg`, `tools/*` — carries the same comment shape. **One English line, one Korean line, then a pointer to the document.**

```caddy
# Per-site JSON access log with rolling. Usage: import logsite <hostname>
# 사이트별 JSON 액세스로그, 자동 롤링. 사용: import logsite <hostname>
# refer to docs/logging.md
```

One block of three per thing worth naming: per snippet, per service, per group of variables. The file says what the thing is and how to call it. Everything else — why it works that way, what breaks when you get it wrong, what was measured — goes in `docs/`. A caveat that lives only in a comment is a caveat nobody finds while debugging.

Two exceptions. **Editing instructions may sit inline**, because they are directions rather than explanations, and they are written on one line in both languages:

```caddy
	# Delete this line once the host name is a real domain. / 실제 도메인이면 이 줄을 지운다.
	tls internal
```

And a file **generated** by a tool carries the same three-line header, written by the generator.

## Settled decisions

All of these came from measurement; the evidence is in [docs/caddy-gotchas.md](docs/caddy-gotchas.md). Do not re-open them without new measurements.

- **Block with `respond 444`, not `abort`.** `abort` discards every field added by `log_append`, so the reason never reaches the log. Only places that need no reason (internal-only patterns, the catch-all) use `abort`.
- **The three scanner layers take separate arguments.** Merge them and `probe-ext`'s `*.php` also exempts `probe-app`'s `/wp-login.php`, which opens the app layer.
- **A snippet using `{args[:]}` is dangerous with no argument.** `path` becomes `null` and kills startup, but `remote_ip` becomes an empty object and **matches nothing, with no error anywhere.** Believing you are blocking is the worst state. Any new IP-based snippet must be tried once with no argument.
- **Never switch `network_mode: host` to bridge.** Every `remote_ip` then reads as the Docker gateway and every IP matcher silently stops working.
- **Never use `${VAR:?}` in compose.** Compose interpolates the whole file before it filters profiles, so one `:?` in a disabled service blocks `docker compose up` entirely.
- **Never add `ai-assist-deny` to a public site.** Handing a link to an AI tool for a summary then returns 403.
- **Do not edit `00-*.caddy` directly.** Anyone cloning this repo hits a conflict on `git pull`. Tune through arguments where possible; when new behaviour is genuinely needed, add another snippet under a new name. Address-set files are the exception.
- **The number is ownership, not priority.** `00-` is this repo's namespace; `01-` onward belongs to the deployment. A clone that needs its own snippet adds `conf.d/01-*.caddy` and leaves `00-` alone. This repo goes on shipping new snippets under `00-` — that is where they belong, and every existing one is already there. Measured: a new upstream `00-` file merges cleanly into a clone unless the clone happens to hold the exact same file name, and then the merge stops with an add/add conflict rather than breaking anything quietly. Pick a distinctive name and that is the whole defence.

## Address sets

There are three distinct ways to handle IP ranges. Mixing them puts the same list in two places, or erases the reason something is allowed through.

**① An address set is defined once, in a snippet.** A *named list of ranges* — internal networks, one country, one institution. One `conf.d/00-*.caddy` file per set. Having several sets is normal: `00-acl.caddy` (internal), `00-geo.caddy` (one country) and `00-acl-<org>.caddy` can coexist. Keep it in a snippet even when only one site uses it. **What is forbidden is copying the list into a site file** — do that and one site has the office WiFi range while another does not.

**② What to do with a set is usually the site's call.** When sites legitimately differ, the snippet **defines the matcher and stops there**. `acl-internal` is like that: one site lets internal through unauthenticated, another sends external to login, another hides from external entirely. When there is only one sensible action, the snippet may act — `geo-allow` has no alternative to dropping.

> ⚠ **Only matcher-defining snippets get the `acl-` prefix.** Anything that blocks gets a name that says so, like `geo-allow` or `gate-stub`. If one repo's `acl-internal` defines a matcher and another's also blocks, moving config between them turns an `import` line into a silent no-op. This has happened.

**③ An IP exception for one site lives in that site's file.** A health-check monitor, a partner range, a mail scanner's source range — holes opened because of that site's circumstances. Put them in the shared table and the reason disappears, and sites that never needed it open too.

```caddy
import acl-internal

# Health-check monitor. Cannot do browser auth, so it is exempt.
@healthcheck remote_ip 198.51.100.7/32
```

A range worth naming is ①. A hole that needs a reason written next to it is ③.

## Verifying

- Syntax checking is **`caddy validate`**, not `caddy adapt`. `adapt` never loads the modules, so it misses missing arguments.
- `Caddyfile` is a **file** bind mount: editing it on the host never reaches the container. `conf.d/` is a **directory** mount and does follow. After changing global settings, `docker compose up -d --force-recreate caddy` is required.
- **The development machine (macOS) has neither caddy nor docker.** `sh -n` is about all that runs there. Verify on a real host in a throwaway container, leaving the running config alone.

```sh
docker run --rm --network none -v /tmp/x:/etc/caddy:ro \
  --entrypoint caddy caddy:2 validate --config /etc/caddy/Caddyfile
```

- Before changing a running configuration, **diff the `adapt` output of the old and new configs.** If host matchers, backend upstreams and `forward_auth` URIs are unchanged, routing did not move.
- **Never write that something was run when it was not.** Say plainly what could not be verified.

## Status markers

When work status has to be written down, use these five.

- `TODO` not started
- `DOING` in progress
- `BLOCKED` cannot proceed without user input or external state
- `DONE` complete and confirmed within the current scope
- `DROP` deliberately not doing it
