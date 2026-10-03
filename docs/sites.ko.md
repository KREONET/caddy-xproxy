# 사이트 추가하기

[🇺🇸 English](sites.md) · [🇰🇷 한국어](sites.ko.md)

## 기본 규칙

`conf.d/` 의 파일 하나가 사이트 하나입니다. 파일명은 자유지만(`import conf.d/*.caddy` 가 전부 읽습니다) 호스트명과 맞추면 찾기 쉽습니다.

```sh
cp conf.d/public.example.com.caddy.example conf.d/app.example.com.caddy
vi conf.d/app.example.com.caddy
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

**`.example` 확장자가 붙은 파일은 로드되지 않습니다.** 글롭이 `*.caddy` 라서 `*.caddy.example` 은 안 걸립니다. 예제를 켜려면 확장자만 떼면 되고, 반대로 사이트를 잠시 내리고 싶으면 `.disabled` 같은 걸 붙이면 됩니다.

동봉된 예제:

두 개가 서로 **대비되도록** 짜여 있습니다. 나란히 놓고 보면 어느 줄이 왜 다른지가 드러납니다.

| 파일 | 보여주는 것 |
|--|--|
| `public.example.com.caddy.example` | 공개 사이트, PHP 백엔드(MediaWiki), 업로드 네임스페이스 면제, 검색엔진·AI 요약 허용 |
| `inside.example.com.caddy.example` | 내부 전용, Node 백엔드(Outline), authentik OIDC 게이트, 내부대역 무인증 통과 |

| | `public` | `inside` |
|--|--|--|
| 공개 여부 | 누구나 | 내부대역 + 로그인한 사람 |
| `robots` | `robots-allow` | `robots-deny` |
| `ai-assist-deny` | **안 붙임** (사람이 읽는 공개 위키) | 붙임 |
| `probe-ext` | `*.php *.php/* /images/*` | `/__none__` |
| 인증 | 없음 | `acl-internal` + `forward_auth` |

## 최소 사이트

```caddy
app.example.com {
	import logsite app.example.com
	import secure
	import badbots

	import probe-secret
	import probe-app /__none__
	import probe-ext /__none__

	reverse_proxy http://127.0.0.1:8080
}
```

호스트명이 공인 DNS 에서 이 서버를 가리키면 인증서는 자동으로 붙습니다. 아직 DNS 가 없으면 `tls internal` 을 한 줄 넣어 자체서명으로 띄우고, 나중에 그 줄을 지우면 됩니다.

## import 순서

**차단 스니펫은 다른 `handle` 보다 먼저 import 하세요.** Caddy 가 `handle` 을 경로 특이도로 재정렬하기 때문에, 뒤에 두면 사이트의 catch-all `handle` 이 먼저 잡아버려 차단이 통째로 우회됩니다.

권장 순서입니다.

```caddy
app.example.com {
	import logsite app.example.com   # 1. 로그
	import secure                       # 2. 헤더
	import robots-deny                  # 3. robots.txt
	import badbots                      # 4. UA 차단
	import probe-secret                 # 5. 스캐너 3레이어
	import probe-app /__none__
	import probe-ext /__none__
	import acl-internal                 # 6. 매처 정의 (있으면)

	handle ... {                        # 7. 그 다음이 라우팅
	}
}
```

## 경로별 라우팅

```caddy
wiki.example.com {
	import logsite wiki.example.com
	import secure
	import badbots
	import probe-secret
	import probe-app /__none__
	import probe-ext *.php *.php/* /images/*

	handle /api/* {
		reverse_proxy http://127.0.0.1:9000
	}
	handle /static/* {
		root * /srv/static
		file_server
	}
	handle {
		reverse_proxy http://127.0.0.1:8080
	}
}
```

`handle` 블록들은 상호배타적입니다. 하나가 매칭되면 거기서 끝납니다. 마지막 인자 없는 `handle` 이 catch-all 입니다.

**순서가 보안 경계인 경우** — 예를 들어 "이 경로만은 내부대역이라도 반드시 인증" — 은 `route` 로 감싸서 순서를 고정하세요. `route` 안에서는 작성 순서가 그대로 평가 순서입니다.

```caddy
route {
	handle /admin* {
		route { forward_auth ... }      # 반드시 먼저
	}
	handle @internal {
		reverse_proxy ...
	}
	handle {
		route { forward_auth ... }
	}
}
```

## 라우트 트리를 두 번 쓰게 될 때

내부대역은 무인증, 외부는 로그인 — 이 구조에서 백엔드가 여러 개면 같은 `handle` 묶음을 두 번 쓰게 됩니다. 복사해두면 한쪽만 고치는 날이 옵니다. **그 사이트 파일 안에서 스니펫을 하나 정의**하세요.

```caddy
# 이 사이트에서만 쓰는 라우트 트리.
(app_routes) {
	handle /api/* {
		reverse_proxy http://127.0.0.1:9000
	}
	handle {
		reverse_proxy http://127.0.0.1:8080
	}
}

app.example.com {
	import logsite app.example.com
	...
	import acl-internal

	handle @internal {
		import log-internal
		import app_routes
	}
	handle {
		route {
			forward_auth 127.0.0.1:4180 { ... }
			import log-oidc
			import app_routes
		}
	}
}
```

스니펫 정의는 파일 어디에 있어도 되고 `00-` 접두사가 필요 없습니다 — `(app_routes)` 는 사이트 블록보다 위에만 있으면 됩니다. 공용 스니펫이 아니라 그 사이트의 것이므로 `conf.d/00-*.caddy` 에 올리지 말고 사이트 파일 안에 두세요.

## probe-ext 인자 정하기

백엔드 언어와 업로드 경로 두 가지를 넣습니다. 앱별 표는 [스캐너 차단 설계](scanner-defense.ko.md)에 있습니다.

**업로드 경로를 모르면 추측하지 말고 관찰 모드로 먼저 돌리세요.**

```caddy
import probe-ext-observe *.php *.php/*
```

1~2주 뒤 로그를 보면 빠뜨린 경로가 드러납니다. `block_reason` 이 이미 붙은 요청은 걸러냅니다 — 앞 레이어가 차단한 것도 `would_block` 태그를 같이 받기 때문에, 그대로 두면 `/wp-login.php` 같은 게 "probe-ext 가 잡을 뻔한 정상 경로"처럼 보입니다.

```sh
jq -r 'select(.would_block) | select(.block_reason | not)
         | "\(.would_block)\t\(.request.uri)"' logs/*/access.log \
  | sort | uniq -c | sort -rn | head -50
```

## 내부망 대역

`conf.d/00-acl.caddy` 의 기본값은 RFC1918 사설대역입니다. 자기 환경에 맞게 **그 파일 한 곳만** 고치세요.

```caddy
(acl-internal) {
	@internal remote_ip 10.0.0.0/8 192.0.2.0/24 198.51.100.0/24
}
```

사이트마다 대역표를 복사해두면 반드시 어긋납니다. 어떤 사이트엔 사내 WiFi 대역이 있고 어떤 사이트엔 빠지는 식으로요.

**사이트별 예외**(제휴 대역, 헬스체크 IP 등)는 공용 표에 넣지 말고 그 사이트 파일에서 별도 매처로 붙이세요. 그래야 "왜 이 IP만 통과하는지"가 그 자리에 남습니다.

```caddy
import acl-internal

# 헬스체크 모니터. 브라우저 인증을 못 하는 클라이언트라 예외.
@healthcheck remote_ip 198.51.100.7/32

handle @internal {
	import log-internal
	reverse_proxy http://127.0.0.1:8080
}
handle @healthcheck {
	import log-internal
	reverse_proxy http://127.0.0.1:8080
}
handle {
	route { forward_auth ... }
}
```

## 흔한 실수

**`import probe-ext` 를 인자 없이 쓴다** — `caddy validate` 는 통과하고 **리로드할 때 프록시가 죽습니다.** 해당 없으면 `/__none__` 을 명시하세요.

**자바 백엔드에 `*.action` 을 안 넣는다** — Confluence 가 즉시 죽습니다. `.do` 와 `.action` 은 자바 웹앱의 정상 확장자입니다.

**`conf.d/` 를 비운다** — 에러 없이 아무것도 안 하는 프록시가 됩니다.

**`00-*.caddy` 스니펫을 직접 고친다** — `git pull` 때 충돌합니다. 사이트 파일에서 인자로 조절하는 쪽이 낫습니다. 대역표(`00-acl.caddy`)는 예외로, 원래 각자 고쳐 쓰는 파일입니다.

**공개 위키에 `ai-assist-deny` 를 붙인다** — 연구자가 AI 도구에 링크를 주고 요약을 시키면 403 이 납니다. 사람이 읽으라고 만든 곳에는 붙이지 마세요.
