# 스캐너 차단 설계

[🇺🇸 English](scanner-defense.md) · [🇰🇷 한국어](scanner-defense.ko.md)

## 명제

> [!IMPORTANT]
> **(기본 명제)** 이 사이트에 없는 것을 찾는 요청은 방문이 아니라 정찰이다.

리버스프록시는 뒷단에 뭐가 있는지 알고 있습니다. Tomcat 이 돌고 있다는 걸 아는데 `.php` 요청이 들어오면, 그건 잘못 찾아온 사용자가 아니라 뭐가 있는지 훑고 있는 스캐너입니다. 확률이 아니라 확정입니다 — 백엔드가 PHP 를 실행할 수 없으니 그 요청에 정상적인 의도가 존재할 수 없습니다.

이게 UA 기반 봇 차단과 다른 점입니다. UA 는 자진신고에 의존하니까 속이면 그만이지만, "네 백엔드가 뭘 실행할 수 있느냐"는 스캐너가 바꿀 수 없습니다.

## 왜 한 번이면 충분한가

일반적인 차단은 "N번 실패하면 밴" 입니다. 실패가 정상일 수도 있으니까요 — 비밀번호는 틀릴 수 있습니다.

그런데 위 요청들은 **정상일 수가 없습니다.** 그래서 임계값이 필요 없습니다. 한 번 훑으면 그 자리에서 판정됩니다. 실제 프록시 로그에서 확인한 사례를 보면, `.php` 를 훑는 IP 는 같은 세션에서 `/wp-config.php`, `/.ssh/id_rsa`, `/backup.sql`, `/_vti_pvt/service.pwd` 를 차례로 찍고 지나갑니다. 두 번째를 기다릴 이유가 없습니다.

## 3레이어

하나의 명제지만 **예외를 선언하는 주체가 달라서** 레이어를 나눕니다.

| 레이어 | 묻는 것 | 사이트별 예외 |
|--|--|--|
| `probe-secret` | 비밀·크리덴셜 파일인가 | **없음** |
| `probe-app` | 여기 없는 *앱* 의 지문인가 | 내가 그 앱일 때의 정상 경로 |
| `probe-ext` | 여기 없는 *언어* 의 확장자인가 | 내 언어 + 업로드 네임스페이스 |

### ① probe-secret

`/.env`, `/.ssh/id_rsa`, `/.git/config`, `/.aws/credentials`, `/rclone.conf` 같은 것들입니다. 어떤 웹앱도 이걸 정상 기능으로 서빙하지 않으므로 사이트별 예외가 없습니다.

```caddy
import probe-secret
```

> ⚠ 점파일 패턴을 추가할 때 `/.well-known/*` 을 잡지 않는지 확인하세요. ACME 인증서 갱신과 OIDC discovery 가 그리로 옵니다.

### ② probe-app

`/wp-login.php`, `/wp-content/*`, `/phpmyadmin/`, `/xmlrpc.php` — **여기 없는 앱**의 지문입니다. 지금 WordPress 를 안 돌린다고 영원히 안 돌리는 건 아니니까, 이건 사이트별로 뒤집을 수 있어야 합니다.

```caddy
# WordPress 를 안 돌린다
import probe-app /__none__

# WordPress 를 돌린다 — 그 사이트에서만 정상 경로가 된다
import probe-app /wp-admin/* /wp-login.php /wp-content/* /wp-includes/* /xmlrpc.php
```

### ③ probe-ext

백엔드가 실행할 수 없는 확장자입니다. 인자는 두 종류를 받습니다.

**ⓐ 내 백엔드가 실제로 실행하는 확장자** · **ⓑ 사용자 업로드 파일이 놓이는 경로**

ⓑ 가 이 설계에서 제일 중요한 부분입니다.

## 확장자만으로는 부족한 이유

위키에 예제 `.py` 파일을 첨부해두고 사람들이 내려받는다고 합시다. 백엔드는 PHP 고, `.py` 는 실행되지 않습니다. 그런데 그 요청은 **정상입니다** — 실행 요청이 아니라 다운로드니까요.

확장자는 "실행 요청"과 "콘텐츠 다운로드"를 구분하지 못합니다. 구분해주는 건 **경로**입니다. 업로드 파일은 앱마다 정해진 네임스페이스 아래에만 놓입니다.

```caddy
import probe-ext *.php *.php/* /images/*
```

```
/images/example.py        통과    ← 위키에 올린 예제 파일
/images/nested/a/b.pl     통과    ← 하위 디렉터리까지 재귀
/shell.py                 444     ← 루트에 던진 프로브
/admin.jsp                444
```

## 앱별 선언표

| 앱 | 백엔드 | `import probe-ext` 인자 |
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
| NetBox · Django 계열 | Python | `/media/*` |
| Mattermost · Outline · Node/Go 일반 | — | `/__none__` |

> ⚠ `*.do` 와 `*.action` 은 자바 웹앱(Struts, Confluence)의 **정상** 확장자입니다. 자바 백엔드라면 반드시 인자에 넣으세요. 안 넣으면 사이트가 즉시 죽습니다.

> ⚠ 업로드 네임스페이스를 **추측하지 마세요.** 관찰 모드로 돌리면 로그가 알려줍니다(아래).

## 확장자 없는 요청은 평가하지 않습니다

이게 이 레이어가 안전한 이유입니다. 웹앱 정상 트래픽의 대부분은 확장자가 없습니다.

```
/rest/api/2/issue                      평가 안 함
/api/v4/users                          평가 안 함
/plugins/servlet/gadgets               평가 안 함
/s/abc-CDN/of23ld/9422/batch.js        평가 안 함
```

mod_rewrite 로 확장자가 사라진 URL 도 마찬가지입니다. rewrite 는 이 설계에 **유리한 방향**으로만 작동합니다 — 확장자를 지우면 평가 대상에서 빠지고, 확장자를 허용하면 인자에 넣으면 됩니다.

## 관찰 모드 — 반드시 여기부터

차단하지 않고 `would_block` 태그만 기록합니다. 밴까지 연결한 상태에서 업로드 네임스페이스를 하나 빠뜨리면 실사용자가 잘려나가므로, 1~2주 돌려서 오탐 0 을 확인한 뒤 전환하세요.

```caddy
import probe-secret
import probe-app-observe /__none__
import probe-ext-observe *.php *.php/*
```

```sh
# would_block 이 찍힌 요청만 뽑아 본다. 앞 레이어가 이미 차단한 것(block_reason)은 뺀다 —
# 한 요청이 두 레이어에 다 걸릴 수 있어서, 안 빼면 차단된 프로브가 오탐 후보처럼 보인다.
jq -r 'select(.would_block) | select(.block_reason | not)
         | "\(.would_block)\t\(.request.client_ip)\t\(.request.uri)"' \
  logs/*/access.log | sort | uniq -c | sort -rn | head -50
```

여기 나온 것 중 **정상 트래픽이 섞여 있으면 그게 곧 빠뜨린 네임스페이스**입니다. 인자에 추가하고 다시 관찰합니다. 목록이 스캐너만 남으면 `-observe` 를 떼고 차단으로 바꿉니다.

이미 운영 중인 프록시에 도입한다면 과거 로그로 같은 검증을 즉시 할 수 있습니다. 축적된 액세스로그의 고유 경로 전부를 새 매처에 통과시켜서, 걸리는 것 중 정상이 있는지만 보면 됩니다.

## 차단 방식이 `respond 444` 인 이유

`abort` 는 연결을 끊으면서 `log_append` 로 붙인 필드를 **통째로 날립니다**. 실측 결과:

| 방식 | 응답 | `block_reason` |
|--|--|--|
| `log_append` + `abort` | 연결끊김 | **유실** |
| `log_append` + `respond 403` | 403 | 기록 |
| `log_append` + `error 403` | 403 | 기록 |
| `log_append` + `respond 444` | 444 (빈 본문) | 기록 |

차단 사유가 로그에 안 남으면 fail2ban 이 경로 목록을 regex 로 복제해야 하고, 그 순간 두 곳이 어긋나기 시작합니다. `444` 는 본문 없는 비표준 코드라 스캐너가 얻는 정보는 `abort` 와 사실상 같습니다. 근거는 [Caddy 함정 모음](caddy-gotchas.ko.md) 문서에 정리해 두었습니다.

## 레이어마다 인자를 따로 받는 이유

세 레이어를 하나의 인자 목록으로 합치면 뚫립니다. 실측으로 확인한 사례입니다.

MediaWiki 사이트가 `*.php` 를 "내 언어"로 선언합니다. 인자를 공유하면 그 선언이 `probe-app` 에도 적용돼서 `/wp-login.php` 가 `*.php` 에 매칭되어 **면제**됩니다. WordPress 지문 레이어가 통째로 무력화됩니다.

```
인자 공유:  MediaWiki 사이트 → /wp-login.php → 통과 200   ✗
레이어 분리: MediaWiki 사이트 → /wp-login.php → 444        ✓
```

## fail2ban 연계

이 레포는 **탐지와 로깅까지** 담당합니다. 밴은 별도입니다.

```
block_reason = probe-secret | probe-app | probe-ext     ← 1-strike 대상
             = ai-training-crawler | empty-user-agent   ← UA 기반, 정책에 따라
             = ai-assistant-fetch
```

`ignoreip` 에 내부 대역을 반드시 넣으세요. 취약점 점검 스캐너를 사내에서 돌리면 자기 자신을 밴합니다. 로그 스키마는 [로그](logging.ko.md) 참고.
