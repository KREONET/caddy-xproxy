# caddy-xproxy — 바로 쓰는 Caddy 리버스프록시 스택

[🇺🇸 English](README.md) · [🇰🇷 한국어](README.ko.md)

스캐너·봇 차단, OIDC 게이트 등 유용한 스니펫이 기본으로 들어가 있는 리버스프록시입니다.

## 설치

```sh
git clone https://github.com/kreonet/caddy-xproxy /opt/xproxy
cd /opt/xproxy
cp .env.example .env
cp conf.d/public.example.com.caddy.example conf.d/public.example.com.caddy
docker compose up -d
```

동봉된 샘플 사이트의 `.example` 만 떼면 클론 직후에도 바로 응답합니다.

```sh
curl -k --resolve public.example.com:443:127.0.0.1 https://public.example.com/
# caddy-xproxy is running. Replace this block with your own site.
```

자세한 내용은 [설치](docs/install.ko.md) 문서를 참고하세요.

## 요구사항

* Docker + Docker Compose
* 80 / 443 포트가 비어 있어야 합니다. 이 스택은 `network_mode: host` 로 동작합니다
* 공인 인증서를 받으려면 도메인이 이 서버를 가리키고 80 포트가 열려 있어야 합니다 ([TLS](docs/tls.ko.md))

## 사이트 추가

`conf.d/` 에 파일 하나를 만들면 그게 사이트입니다. 동봉된 샘플을 복사해서 고치는 게 가장 빠릅니다.

```sh
cp conf.d/public.example.com.caddy.example conf.d/app.example.com.caddy
vi conf.d/app.example.com.caddy
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

```caddy
app.example.com {
	import logsite app.example.com
	import secure
	import badbots

	import probe-secret
	import probe-app /__none__
	import probe-ext *.php *.php/* /uploads/*

	reverse_proxy http://127.0.0.1:8080
}
```

예제는 둘입니다. `public.example.com.caddy.example`(공개 + PHP + 업로드 네임스페이스)과 `inside.example.com.caddy.example`(내부 전용 + Node + OIDC 게이트). `.example` 이 붙은 파일은 로드되지 않으니 확장자만 떼면 켜집니다. 자세한 내용은 [사이트 추가하기](docs/sites.ko.md) 문서를 참고하세요.

## 미리 준비된 유용한 스니펫

| import | 하는 일 |
|--|--|
| `logsite <host>` | 사이트별 JSON 액세스로그, 자동 롤링 |
| `secure` · `secure-embed` | HSTS·CSP 등 보안 헤더 부착, 서버 정보 노출 숨김 |
| `probe-secret` · `probe-app` · `probe-ext` | 스캐너 탐지 3레이어 |
| `badbots` · `ai-assist-deny` · `badbots-ua-ok` | 크롤러 차단 (User-Agent 기반) |
| `robots-allow` · `robots-deny` | robots.txt 를 Caddy 가 직접 응답 |
| `acl-internal` | 내부망 대역 매처 (표는 한 곳에만) |
| `geo-allow` | 특정 국가 밖 접근 차단 (목록은 생성) |
| `gate-stub` | OIDC 를 완주 못하는 자동화 클라이언트 종결 |
| `log-oidc` · `log-internal` | 액세스로그에 "누가" 기록 |
| `dns01` | DNS-01 인증서 발급 |

스니펫별 인자와 주의사항은 [스니펫 레퍼런스](docs/snippets.ko.md) 문서에 정리해 두었습니다.

## 스캐너 차단

> [!IMPORTANT]
> **(기본 명제)** 이 사이트에 없는 것을 찾는 요청은 방문이 아니라 정찰이다.

Tomcat 백엔드에 `.php` 를 묻는 요청, WordPress 를 안 돌리는데 `/wp-login.php` 를 찾는 요청, 어떤 앱도 서빙하지 않는 `/.env` 를 찾는 요청. 셋 다 한 번이면 충분한 증거입니다. 각 사이트가 "나는 이런 걸 가지고 있다"를 한 줄로 선언하면, 나머지는 전부 정찰로 처리됩니다.

확장자 없는 요청(`/api/v4/users`, SPA 라우트, rewrite 된 URL)은 아예 평가하지 않고, 위키에 올려둔 예제 `.py` 파일 같은 건 업로드 경로를 선언해 면제합니다.

차단 사유는 액세스로그에 `block_reason` 으로 남으므로 fail2ban 이 그대로 활용할 수 있습니다. 처음 도입할 때는 차단 없이 태그만 남기는 관찰 모드를 먼저 돌려 보세요. 설계 근거는 [스캐너 차단 설계](docs/scanner-defense.ko.md) 문서에 있습니다.

## OIDC 게이트

백엔드에 로그인이 없거나, 있어도 프록시 단에서 먼저 막고 싶을 때 사용합니다. 내부망은 무인증으로 통과시키고, 외부는 IdP 로그인으로 보냅니다.

```sh
cp oauth2-proxy/oidc.cfg.example oauth2-proxy/oidc.cfg
cp conf.d/inside.example.com.caddy.example conf.d/inside.example.com.caddy
docker compose --profile oidc up -d
```

설정 절차는 [OIDC 게이트](docs/oidc.ko.md) 문서를 참고하세요.

## 문서

| | |
|--|--|
| [설치](docs/install.ko.md) | 요구사항, 첫 기동, 리로드, 문제 해결 |
| [사이트 추가하기](docs/sites.ko.md) | conf.d 규칙, 라우팅, 흔한 실수 |
| [스니펫 레퍼런스](docs/snippets.ko.md) | import 하나하나의 인자와 주의사항 |
| [스캐너 차단 설계](docs/scanner-defense.ko.md) | 3레이어, 앱별 선언표, 관찰 모드 |
| [OIDC 게이트](docs/oidc.ko.md) | oauth2-proxy, forward_auth, 내부 우회 |
| [TLS](docs/tls.ko.md) | HTTP-01, DNS-01, 자체서명 |
| [로그](docs/logging.ko.md) | JSON 스키마, block_reason, fail2ban 연계 |
| [Caddy 함정 모음](docs/caddy-gotchas.ko.md) | 실측으로 확인한 것들 |

## 라이선스

MIT
