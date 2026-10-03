# TLS

[🇺🇸 English](tls.md) · [🇰🇷 한국어](tls.ko.md)

Caddy 는 인증서를 알아서 받습니다. 사이트 블록에 호스트명만 적혀 있으면 기본적으로 Let's Encrypt 에서 HTTP-01 으로 발급받습니다. 설정이 필요한 건 그게 안 되는 경우뿐입니다.

## 어느 방식을 쓸까

| 상황 | 방식 | 설정 |
|--|--|--|
| 도메인이 이 서버를 가리키고 80 포트가 공인망에서 열림 | HTTP-01 | **없음** (기본) |
| 80 포트가 공인망에서 도달 불가 | DNS-01 | `import dns01` + 플러그인 빌드 |
| 도메인이 아직 없거나 내부 테스트 | 자체서명 | `tls internal` |

## HTTP-01 (기본)

아무것도 안 하면 이겁니다.

```caddy
app.example.com {
	import logsite app.example.com
	reverse_proxy http://127.0.0.1:8080
}
```

조건 두 가지입니다. `app.example.com` 의 A/AAAA 레코드가 이 서버를 가리킬 것, 그리고 Let's Encrypt 가 **80 포트로 직접 들어올 수 있을** 것. 방화벽이 80 을 막고 있으면 실패합니다.

`.env` 의 `ACME_EMAIL` 을 채우면 만료 경고 메일을 받습니다.

## 자체서명

도메인이 아직 준비 안 됐거나 내부에서만 쓸 때입니다.

```caddy
app.example.com {
	tls internal
	...
}
```

브라우저는 경고를 냅니다. `curl` 은 `-k` 가 필요합니다. 나중에 공인 인증서로 옮길 때는 `tls internal` 줄만 지우면 됩니다.

동봉된 `conf.d/public.example.com.caddy.example` 이 이 방식입니다. `example.com` 은 문서용으로 예약된 도메인(RFC 2606)이라 이 서버를 가리킬 수 없고, 따라서 ACME 발급도 되지 않습니다.

## DNS-01

80 포트가 공인망에서 도달 불가일 때 사용합니다. 내부망 전용 호스트나 방화벽 뒤가 여기 해당합니다. 인증서 검증을 DNS 레코드로 하므로 인바운드 연결이 필요 없습니다.

이 레포는 RFC2136(동적 DNS 업데이트) 설정을 동봉합니다.

### ① 플러그인이 든 이미지 빌드

스톡 `caddy:2` 에는 DNS 플러그인이 없습니다.

```sh
vi docker-compose.yml
```

```yaml
    # image: caddy:2
    build: ./caddy
```

```sh
docker compose up -d --build
```

다른 DNS 제공자(Cloudflare, Route53 등)를 쓴다면 `caddy/Dockerfile` 의 `--with` 를 바꾸세요. 목록은 <https://github.com/caddy-dns> 에 있습니다. 그 경우 `conf.d/00-tls-dns01.caddy` 의 `dns rfc2136 { ... }` 블록도 해당 제공자 문법으로 바꿔야 합니다.

### ② DNS 위임

`_acme-challenge.<호스트명>` 이 TSIG 업데이트를 받는 네임서버로 NS 위임돼 있어야 합니다.

### ③ TSIG 키

```sh
vi .env
```

```
DNS_TSIG_KEYNAME=acme-update
DNS_TSIG_ALG=hmac-sha256
DNS_TSIG_SECRET=...
DNS_TSIG_SERVER=ns.example.com:53
```

### ④ 사이트에 적용

```caddy
app.example.com {
	import dns01
	import logsite app.example.com
	reverse_proxy http://127.0.0.1:8080
}
```

> ⚠ 위 셋 중 하나라도 빠지면 발급이 실패하고 Caddy 가 자체서명으로 떨어집니다. 브라우저 경고가 뜨는데 로그를 안 보면 원인을 모릅니다.

```sh
docker compose logs caddy | grep -i acme
```

## 인증서는 어디에 있나

`caddy_data` named volume 입니다. 발급받은 인증서와 **ACME 계정키**가 여기 들어 있습니다.

bind mount 로 바꾸지 마세요. 실수로 지우면 계정키가 날아가고, Let's Encrypt 는 발급 한도가 있습니다.

```sh
docker compose down       # 볼륨 유지. 안전
docker compose down -v    # 볼륨 삭제. 계정키까지 날아간다
```

## 와일드카드

DNS-01 에서만 됩니다. HTTP-01 으로는 발급할 수 없습니다.

```caddy
*.example.com {
	import dns01
	import logsite wildcard.example.com
	...
}
```

와일드카드 사이트 블록은 명시적으로 적힌 호스트명보다 **나중에** 매칭됩니다. `app.example.com` 블록이 따로 있으면 그쪽이 우선입니다.

## 안 될 때

**"could not get certificate"** — DNS 가 이 서버를 가리키는지, 80 포트가 밖에서 열리는지 확인하세요.

```sh
dig +short app.example.com
curl -I http://app.example.com/.well-known/acme-challenge/test
```

**`/.well-known/` 이 차단됨** — 차단 스니펫을 고쳤다면 `/.well-known/*` 이 걸리지 않는지 확인하세요. ACME 갱신과 OIDC discovery 가 그리로 옵니다. 기본 설정은 걸리지 않습니다.

**rate limit** — Let's Encrypt 는 도메인당 주당 발급 횟수 제한이 있습니다. 반복 실패로 한도를 쓰기 전에 스테이징으로 시험하세요.

```caddy
{
	acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
}
```

스테이징 인증서는 브라우저가 신뢰하지 않습니다. 설정이 맞는지 확인하는 용도입니다. 확인 후 이 줄을 지우고 `docker compose down -v` 로 스테이징 계정을 비운 다음 다시 올리세요.
