# Caddy 함정 모음

[🇺🇸 English](caddy-gotchas.md) · [🇰🇷 한국어](caddy-gotchas.ko.md)

이 레포의 설정이 왜 그런 모양인지에 대한 답입니다. 전부 Caddy v2.11.4 에서 직접 재본 것이고, 추측이 아닙니다.

## abort 는 log_append 를 날린다

`abort` 로 차단하면 로그 라인은 남지만(`status: 0`) `log_append` 로 붙인 필드는 사라집니다.

| 방식 | 응답 | `block_reason` |
|--|--|--|
| `log_append` + `abort` | 연결끊김 | **유실** |
| `log_append` + `respond 403` | 403 | 기록 |
| `log_append` + `error 403` | 403 | 기록 |
| `log_append` + `respond 444` | 444 | 기록 |
| 사이트 스코프에서 매처로 태깅 후 별도 `abort` | 연결끊김 | **유실** |

마지막 줄이 중요합니다. `log_append @matcher key value` 를 사이트 스코프에 두고 `abort` 는 다른 handle 에서 하는 우회로도 통하지 않습니다. 차단 사유를 남겨야 하면 `abort` 를 쓸 수 없습니다.

그래서 `00-probe.caddy` 와 `00-badbots.caddy` 는 `respond` 로 처리합니다. 사유를 안 남기는 곳(`00-acl.caddy` 의 내부전용 패턴, Caddyfile 의 catch-all)만 `abort` 입니다.

## 인자 없이 import 하면 adapt 는 통과하고 기동이 죽는다 (`path` 의 경우)

`{args[:]}` 를 쓰는 스니펫에 인자를 안 주면 이렇게 됩니다.

```sh
$ caddy adapt    --config Caddyfile   # 통과. "not":[{"path":null}] 로 나온다
$ caddy validate --config Caddyfile   # 여기서 걸린다
$ caddy run      --config Caddyfile
Error: ... provision http.matchers.not: loading matcher sets:
       module name 'path': module value cannot be null
```

`adapt` 는 Caddyfile 을 JSON 으로 바꾸기만 하고 모듈을 실제로 적재하지 않습니다. `validate` 는 provision 단계까지 돌려서 이걸 잡습니다. **배포 스크립트에서는 `adapt` 말고 `validate` 를 쓰세요.**

이 레포의 `probe-app` · `probe-ext` 가 해당 없음을 `/__none__` 으로 **명시**하게 돼 있는 이유입니다. 경로는 `/` 로 시작하므로 `/__none__` 은 아무것도 매칭하지 않습니다.

## 인자 없는 `remote_ip` 는 죽지도 않고 매칭도 안 한다

위의 `path` 와 반대입니다. `{args[:]}` 를 쓰는 스니펫에 인자를 안 줬을 때, 매처가 무엇이냐에 따라 결과가 갈립니다.

| 매처 | 인자 없을 때 adapt 결과 | 기동 |
|--|--|--|
| `path {args[:]}` | `{"path":null}` | **죽는다** |
| `remote_ip {args[:]}` | `{"remote_ip":{}}` | **뜬다. 그리고 아무것도 매칭 안 한다** |

실측(Caddy v2.11.4). `import gate-stub` 를 인자 없이 쓰고 `:8081` 에 붙여 요청하면 스텁이 아니라 뒤의 핸들러가 응답합니다.

```sh
8081 (인자 없음)          → PASSTHROUGH   ← 스니펫이 통째로 무효
8082 (인자 있음, 대역 밖) → PASSTHROUGH
8083 (인자 있음, 대역 안) → 스텁 응답
```

`path` 쪽은 틀리면 프록시가 안 떠서 바로 압니다. `remote_ip` 쪽은 **막고 있다고 믿는 상태로 계속 돕니다.** IP 기반 예외·차단 스니펫을 새로 쓸 때는 인자를 뺀 경우를 한 번 때려보고 의도대로 동작하는지 확인하세요.

## Caddyfile 을 호스트에서 고쳐도 컨테이너는 못 본다

`conf.d/` 는 **디렉터리** 마운트라 호스트에서 고친 게 바로 보입니다. `Caddyfile` 은 **파일** 마운트라 안 보입니다.

```yaml
- ./Caddyfile:/etc/caddy/Caddyfile:ro   # 파일 — inode 에 고정된다
- ./conf.d:/etc/caddy/conf.d:ro         # 디렉터리 — 안의 변화가 따라온다
```

도커는 파일을 마운트할 때 그 **inode** 를 붙입니다. 편집기는 저장할 때 대개 새 파일을 쓰고 이름을 바꿔 덮으므로(`vi`, `sed -i`, `cp`, `mv` 전부) 호스트 쪽 inode 가 바뀌고, 컨테이너는 **옛 inode 를 계속 봅니다.**

```sh
$ stat -c %i Caddyfile                                   # 255164
$ docker compose exec caddy stat -c %i /etc/caddy/Caddyfile   # 252476
```

그래서 `Caddyfile` 을 고친 뒤 `caddy reload` 를 하면 **옛 내용이 다시 로드됩니다.** 에러가 없어서 반영된 줄 알게 됩니다. 컨테이너를 다시 만들어야 합니다.

```sh
docker compose up -d --force-recreate caddy
```

사이트를 더하거나 고치는 일은 전부 `conf.d/` 에서 일어나므로 평소에는 `reload` 로 충분합니다. `Caddyfile` 자체를 건드리는 건 전역 옵션이나 catch-all 을 바꿀 때뿐이고, 그때만 재생성이 필요합니다.

## conf.d 가 비면 에러 없이 빈 설정으로 뜬다

```sh
$ caddy adapt --config Caddyfile      # import conf.d/*.caddy 가 0개 매칭
{"admin":{"disabled":true}}           # 에러가 아니다
```

프록시는 정상 기동하고 아무 일도 하지 않습니다. 원인이 로그에 남지 않아서 "떴는데 왜 안 되지"로 시간을 허비하게 됩니다. 그래서 `conf.d/public.example.com.caddy.example` 을 동봉합니다 — `.example` 만 떼면 곧바로 응답하는 사이트가 하나 생기므로, 설치 직후 스택이 살아 있는지 확인하는 기준이 됩니다.

## path 매처의 `*` 는 빈 문자열도 매칭한다

```
@x path */.env*
   /.env                  BLOCK      ← 앞의 * 가 빈 문자열
   /sub/.env              BLOCK
   /a/b/c/.env.local      BLOCK
```

반대로 루트형은 하위 경로를 못 잡습니다.

```
@b path /.env*
   /.env                  BLOCK
   /sub/.env              통과       ← 못 잡는다
```

`*/x*` 한 형태가 루트와 하위를 모두 덮으므로 `/x*` 와 `*/x*` 를 쌍으로 쓸 필요가 없습니다.

## path 매처는 대소문자를 무시한다

```
@c path /phpmyadmin*
   /phpMyAdmin/index.php  BLOCK
   /PHPMYADMIN/           BLOCK
```

`/phpMyAdmin*` 같은 변형 표기를 따로 넣을 필요가 없습니다. 그리고 경로는 매칭 전에 정규화되므로 `/x.jspa/../y.php` 같은 traversal 로 빠져나갈 수 없습니다. 쿼리스트링은 경로에 포함되지 않습니다.

## handle 은 재정렬되고 route 는 순서를 지킨다

Caddy 는 `handle` 블록을 경로 특이도로 재정렬합니다. 작성 순서가 곧 평가 순서가 아닙니다.

순서가 보안 경계인 경우 — 예를 들어 "이 경로만은 내부대역이라도 반드시 인증" — `route` 로 감싸서 순서를 손으로 고정하세요.

```caddy
route {
	handle /admin* {         # 이게 먼저인 게 보장된다
		route { forward_auth ... }
	}
	handle @internal {
		reverse_proxy ...
	}
	handle {
		route { forward_auth ... }
	}
}
```

같은 이유로 `log-oidc` 는 반드시 `route` 안에서 `forward_auth` **다음 줄**에 와야 합니다. `handle` 만 쓰면 재정렬돼서 헤더가 붙기 전에 실행되고, `auth_email` 이 빈 값으로 남습니다.

## 스니펫이 handle 로 감싸져 있는 이유

Caddy 의 지시어 순서상 `handle` 이 `respond` 나 `abort` 보다 먼저 평가됩니다. 그래서 맨몸 `respond` 로 만든 차단 스니펫은 **`handle` 블록을 쓰는 사이트에서 통째로 우회됩니다.**

```caddy
# 이렇게 쓰면 안 된다 — handle 을 쓰는 사이트에서 무력화된다
(badbots) {
	@bot header_regexp User-Agent (?i)GPTBot
	respond @bot 403
}
```

그래서 이 레포의 차단 스니펫은 전부 `handle` 로 감싸져 있고, **사이트 블록 안에서 다른 handle 보다 먼저 import** 해야 합니다.

## 스니펫의 named matcher 는 사이트 스코프에서 동작한다

스니펫 안에서 정의한 named matcher 는 사이트 블록에 import 하면 그 사이트의 `route` · `handle` 안 어디서든 참조할 수 있습니다. `00-acl.caddy` 가 `@internal` 을 이렇게 제공합니다.

단 **import 는 사이트 블록 최상단에** 둬야 합니다. `route` 나 `handle` 안에서 import 하면 매처가 그 안에 갇힙니다.

## {args[:]} 는 매처 위치에서 동작한다

가변 인자는 매처 위치에서 정상 확장됩니다.

```caddy
(alien) {
	@a {
		path *.php *.jsp *.asp
		not path {args[:]}
	}
	...
}
```

다만 `header` 지시어의 **값** 위치에서는 치환되지 않습니다. 그래서 `00-secure.caddy` 의 `secure-embed` 는 `{args[0]}` 을 쓰고, 값이 여러 개면 따옴표로 묶어 한 인자로 넘기게 돼 있습니다.

## 지시어 순서상 abort 는 reverse_proxy 보다 먼저다

```caddy
site.example {
	import acl-internal
	abort @not_internal       # 이게 먼저 실행된다
	reverse_proxy http://backend
}
```

파일에 쓴 순서와 무관하게 그렇습니다. 다만 이 순서에 의존하는 것보다 `handle @internal { ... } handle { abort }` 로 명시하는 편이 읽는 사람에게 분명합니다. 이 레포의 예제들은 그렇게 돼 있습니다.

## User-Agent 헤더가 "빈 값"이면 통과한다

`not header User-Agent *` 는 헤더가 **없는** 요청을 잡습니다. 헤더는 보내면서 값만 빈 문자열이면 통과합니다 — `header <name> *` 이 "필드가 존재하는가"를 보기 때문입니다.

```
UA 헤더 자체가 없음     → 403
UA: (빈 값)             → 통과
```

실제 스캐너는 대부분 UA 를 아예 안 보내거나 자기 이름을 박으므로 실용상 문제는 거의 없습니다. 빈 값까지 막으려면 `@empty header_regexp User-Agent ^$` 를 따로 추가하세요.

## remote_ip 는 network_mode: host 가 아니면 무의미하다

bridge 네트워크 + 포트매핑으로 띄우면 모든 요청의 `remote_ip` 가 도커 게이트웨이 IP 로 보입니다. `acl-internal` 을 포함한 모든 IP 기반 매처가 통째로 무력화되고, **에러는 나지 않습니다** — 그냥 전부 내부이거나 전부 외부로 판정됩니다.

이 레포의 `docker-compose.yml` 이 `network_mode: host` 인 첫 번째 이유입니다.

## import 경로에 환경변수를 쓸 수 있다

```caddy
import sites/{$INSTANCE}/*.caddy
```

한 레포를 여러 인스턴스가 클론해서 사이트 정의만 갈라 쓰는 구조가 가능합니다. 다만 `INSTANCE` 가 비면 위의 "conf.d 가 비면" 항목과 똑같이 **조용히 빈 설정**이 되므로, 쓴다면 값이 없을 때 멈추는 장치를 따로 두세요. 이 레포는 단순함을 택해 `conf.d/*.caddy` 한 곳만 읽습니다.
