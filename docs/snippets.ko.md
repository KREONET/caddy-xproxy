# 스니펫 레퍼런스

[🇺🇸 English](snippets.md) · [🇰🇷 한국어](snippets.ko.md)

`conf.d/00-*.caddy` 가 제공하는 import 목록입니다. 파일명 앞의 `00-` 은 정렬 순서를 강제하기 위한 것입니다 — 스니펫이 사이트보다 먼저 로드돼야 사이트에서 쓸 수 있습니다.

## logsite

```caddy
import logsite <hostname>
```

사이트별 JSON 액세스로그. `logs/<hostname>/access.log` 에 쌓이고 50MiB 마다 롤, 14개 + 90일 보관합니다.

사이트마다 파일을 나누는 이유는 한 사이트의 스캐너 폭주가 다른 사이트 로그를 밀어내지 않게 하기 위해서입니다. 스키마는 [로그](logging.ko.md) 문서에 정리해 두었습니다.

## secure · secure-embed

```caddy
import secure
import secure-embed https://embed.example.com
import secure-embed "https://a.example https://b.example"
```

CSP, HSTS, `X-Content-Type-Options`, `X-XSS-Protection` 을 붙이고 `Server` 헤더를 지웁니다. `secure-embed` 는 외부 사이트의 iframe 임베드를 허용해야 할 때 `frame-ancestors` 만 넓힌 판입니다.

> ⚠ 허용처가 여러 개면 **따옴표로 묶어 한 인자로** 넘기세요. `header` 지시어의 값 위치에서는 `{args[:]}` 가 치환되지 않아 `{args[0]}` 을 쓰고 있습니다.

## probe-secret · probe-app · probe-ext

```caddy
import probe-secret
import probe-app /__none__
import probe-ext *.php *.php/* /images/*
```

스캐너 탐지 3레이어. 관찰 모드 판도 있습니다.

```caddy
import probe-app-observe /__none__
import probe-ext-observe *.php *.php/*
```

설계 근거, 앱별 인자표, 도입 절차는 전부 [스캐너 차단 설계](scanner-defense.ko.md)에 있습니다. 여기서는 주의사항만.

> ⚠ 인자를 아예 생략하면 `caddy validate` 는 통과하고 **기동할 때 죽습니다.** 해당 없으면 `/__none__` 을 명시하세요.

> ⚠ 세 레이어의 인자를 하나로 합치지 마세요. `*.php` 선언이 `/wp-login.php` 까지 면제해서 앱 레이어가 뚫립니다.

> ⚠ 자바 백엔드는 `*.action` `*.do` 를 반드시 인자에 넣으세요.

> ⚠ 다른 `handle` 보다 먼저 import 하세요.

차단은 `respond 444` 로 합니다. `abort` 는 `log_append` 를 날려서 차단 사유가 로그에 안 남습니다.

## badbots · ai-assist-deny

```caddy
import badbots
import ai-assist-deny
```

User-Agent 기반 차단. `badbots` 는 학습데이터 수집·대량 스크래핑 크롤러와 UA 없는 요청을 403 으로 막습니다. `ai-assist-deny` 는 사람이 AI 도구에 링크를 주고 조회시킬 때 발생하는 요청을 막습니다.

> ⚠ **공개 위키처럼 사람이 읽으라고 만든 곳에는 `ai-assist-deny` 를 붙이지 마세요.** 연구자가 링크를 요약시키면 403 이 나고, 어느 AI 를 쓰느냐에 따라 되고 안 되고가 갈립니다.

UA 는 자진신고라서 속이면 그만입니다. 신고 안 하는 스캐너는 `probe-*` 가 맡습니다. 둘은 서로를 대체하지 않습니다.

`block_reason` 값: `ai-training-crawler`, `empty-user-agent`, `ai-assistant-fetch`.

### badbots-ua-ok

```caddy
import badbots-ua-ok
```

`badbots` 에서 **빈 UA 차단만 뺀** 판입니다. 학습 크롤러 차단은 그대로입니다.

기계가 읽으라고 공개한 데이터 — XML·JSON 피드, 외부 검증기가 가져가는 파일 — 를 두는 사이트에만 쓰세요. PHP 의 `file_get_contents()` 와 `DOMDocument::load()` 는 User-Agent 헤더를 **아예 보내지 않습니다.** 그래서 그런 수집기가 `badbots` 의 빈 UA 매처에 403 으로 막히는데, 수집기 쪽에는 "파일이 비었거나 접근 불가" 로 보고되고 이쪽 로그에는 `empty-user-agent` 만 남아서 양쪽 다 원인이 안 보입니다.

## robots-allow · robots-deny

```caddy
import robots-allow
import robots-deny
```

`robots.txt` 를 Caddy 가 직접 응답합니다. 백엔드가 죽어도, SPA 가 404 자리에 HTML 을 뱉어도 의도한 내용이 나갑니다.

`badbots` 와는 별개 정책입니다. robots.txt 는 권고라서 지키는 크롤러(검색엔진)에게만 의미가 있고, 안 지키는 놈은 `badbots` 가 막습니다.

## acl-internal

```caddy
import acl-internal
```

`@internal` 매처를 **정의만** 합니다. 뭘 할지는 사이트가 정합니다.

```caddy
# 내부는 통과, 외부는 로그인
handle @internal { import log-internal
                   reverse_proxy ... }
handle           { route { forward_auth ... } }

# 내부는 통과, 외부는 연결종료
handle @internal { import log-internal
                   reverse_proxy ... }
handle           { abort }
```

대역표는 `conf.d/00-acl.caddy` 한 곳에만 둡니다. 기본값은 RFC1918 이라 자기 환경에 맞게 고쳐야 합니다.

> ⚠ 사이트 블록 **최상단**에 import 하세요. `route` 나 `handle` 안에서 import 하면 매처가 그 안에 갇힙니다.

> ⚠ `remote_ip` 가 진짜 클라이언트 IP 를 보려면 compose 가 `network_mode: host` 여야 합니다. bridge 로 바꾸면 이 ACL 이 조용히 무력화됩니다.

## log-oidc · log-internal

```caddy
route {
	forward_auth 127.0.0.1:4180 { ... }
	import log-oidc
	reverse_proxy ...
}
```

액세스로그에 "누가" 를 기록합니다. `log-oidc` 는 `auth=oidc` 와 함께 `auth_sub`(OIDC sub 클레임, 불변 식별자), `auth_email` 을 남깁니다. `log-internal` 은 `auth=internal-bypass` 를 남겨서, 식별자가 없는 게 정상임을 명시합니다.

> ⚠ **반드시 `route` 안에서 `forward_auth` 다음 줄에** 두세요. `forward_auth` 가 `copy_headers` 로 헤더를 붙인 뒤에 실행돼야 값이 잡힙니다. `handle` 만 쓰면 Caddy 가 재정렬해서 빈 값이 남습니다.

> 미인증 요청에는 이 필드가 안 붙습니다. 401 을 받아 로그인으로 리다이렉트하는 시점에 핸들러 체인이 끝나기 때문입니다. 설정 오류가 아닙니다.

## gate-stub

```caddy
import gate-stub 198.51.100.0/24
```

OIDC 게이트를 완주할 수 없는 자동화 클라이언트를 게이트 **앞에서** 종결시킵니다. 메일 보안 게이트웨이의 URL 검사기가 대표적입니다 — 게이트 뒤의 앱이 보낸 알림메일 속 링크를 피싱 판정하려고 자동으로 열어보는데, 브라우저가 아니라 쿠키와 폼을 다루지 못해 로그인을 끝까지 못 갑니다. 왜 이런 트래픽이 들어오는지는 [OIDC 게이트](oidc.ko.md#알림-메일-안의-링크가-프록시를-두드린다)에 자세히 있습니다.

그냥 흘려보내면 링크 하나마다 `/oauth2/start` → IdP 로그인화면(200) 까지 갔다가 버려지고, state 없는 콜백이 400 을 남깁니다. IdP 에 헛부하가 걸리고 인증 로그가 실패 기록으로 덮입니다.

응답은 로그인 안내 HTML 200 한 번입니다. 백엔드의 "권한없음" 은 보통 302 → 로그인 페이지고 검사기가 최종적으로 받는 건 그 페이지의 200 이라, 그 종착점만 1홉으로 대체합니다. 302 를 흉내내려면 로그인 경로를 앱마다 하드코딩해야 하고(Jira 는 `/login.jsp`, Confluence 는 `/login.action`) 그 경로를 다시 예외처리하지 않으면 리다이렉트 루프가 됩니다.

> ⚠ 다른 `handle` 보다 먼저 import 하세요. `/oauth2/*` 콜백까지 잡아야 합니다.

> ⚠ 인자를 반드시 주세요. **생략해도 아무 데서도 에러가 안 납니다.** `remote_ip` 매처가 빈 객체가 돼서 아무것도 매칭하지 않고, 스니펫이 통째로 무효가 됩니다. 막고 있다고 믿는 상태가 제일 위험합니다. 자세한 내용은 [Caddy 함정 모음](caddy-gotchas.ko.md) 문서를 참고하세요.

차단이 아니라 정상 종결이라 `block_reason` 이 아니라 `auth=gate-stub` 으로 기록합니다.

## geo-allow

```caddy
import geo-allow
```

특정 국가(+ 항상 허용할 내 대역) 밖에서 오는 접근을 `444` 로 끊습니다. 기본 제공은 `conf.d/00-geo.caddy.example` 이고, 목록은 생성해서 채웁니다.

```sh
./tools/geo-gen.sh kr 203.0.113.0/24 198.51.100.0/24 > conf.d/00-geo.caddy
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

첫 인자가 국가코드, 나머지는 그 국가 밖이지만 반드시 통과해야 하는 대역입니다 — 해외 PoP, 파트너망, 모니터링 소스 같은 것들.

> ⚠ **공개 사이트에는 붙이지 마세요.** 해외 출장자, 로밍, 해외 미러가 전부 끊깁니다.

> ⚠ 목록은 생성 시점의 스냅샷입니다. IP 할당은 계속 바뀌므로 주기적으로 다시 생성하세요.

`robots-deny` 와는 다른 층입니다. robots 는 권고고 이건 접근 자체를 막습니다.

## dns01

```caddy
import dns01
```

DNS-01 방식으로 인증서를 발급합니다. 80 포트가 공인망에서 도달 불가일 때 사용합니다.

> ⚠ rfc2136 플러그인이 들어간 이미지가 필요합니다. 스톡 `caddy:2` 에는 포함돼 있지 않습니다. 빌드 방법은 [TLS](tls.ko.md) 문서를 참고하세요.

## 스니펫을 고쳐야 할 때

`00-*.caddy` 를 직접 고치면 `git pull` 때 충돌합니다. 가급적 사이트 파일에서 인자로 조절하세요.

예외는 `00-acl.caddy` 입니다. 대역표는 원래 각자 환경에 맞게 고쳐 쓰는 파일입니다.

정말 스니펫을 바꿔야 하면 새 이름으로 파일을 하나 더 만드는 쪽이 안전합니다. `conf.d/01-my-snippets.caddy` 처럼요 — `00-` 다음에 로드되고, `git pull` 과 충돌하지 않습니다.
