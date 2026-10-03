# 설치

[🇺🇸 English](install.md) · [🇰🇷 한국어](install.ko.md)

## 요구사항

* Docker + Docker Compose
* **80 / 443 포트가 비어 있어야 합니다.** 이 스택은 `network_mode: host` 로 동작하므로 기존 웹서버와 함께 쓸 수 없습니다. Apache 나 nginx 가 이미 점유하고 있다면 먼저 정리하세요.
* 공인 인증서를 받으려면 도메인이 이 서버를 가리키고 80 포트가 공인망에서 도달 가능해야 합니다 ([TLS](tls.ko.md))

`network_mode: host` 를 사용하는 이유는 두 가지입니다. 내부망 백엔드에 직접 도달하기 위해서, 그리고 `remote_ip` 매처가 진짜 클라이언트 IP 를 보기 위해서입니다. bridge + 포트매핑으로 바꾸면 모든 IP 기반 ACL 이 조용히 무력화됩니다.

## 첫 기동

```sh
git clone https://github.com/kreonet/caddy-xproxy /opt/xproxy
cd /opt/xproxy
cp .env.example .env
cp conf.d/public.example.com.caddy.example conf.d/public.example.com.caddy
docker compose up -d
```

`conf.d/` 의 사이트 파일은 전부 `.example` 로 동봉돼 있습니다 — 글롭이 `*.caddy` 라서 확장자를 떼기 전에는 로드되지 않습니다. 위에서 샘플 하나를 켜 두면 설치 직후 스택이 살아 있는지 바로 확인할 수 있습니다.

`.env` 는 처음엔 그대로 둬도 됩니다. `ACME_EMAIL` 만 채우면 실제 인증서를 받을 때 만료 경고를 받을 수 있습니다.

확인:

```sh
curl -k --resolve public.example.com:443:127.0.0.1 https://public.example.com/
# caddy-xproxy is running. Replace this block with your own site.
```

`example.com` 은 문서용 예약 도메인(RFC 2606)이라 이 서버를 가리킬 수 없고, 그래서 자체서명 인증서가 발급됩니다. `-k` 가 필요한 이유입니다.

## 기동은 됐는데 아무 응답이 없을 때

`conf.d/` 에 `*.caddy` 파일이 하나도 없으면 Caddy 는 **에러 없이** 아무것도 하지 않는 설정으로 기동합니다. 동봉된 예제는 전부 `.example` 이라 확장자를 떼지 않으면 이 상태입니다.

```sh
ls conf.d/*.caddy      # 사이트 파일이 하나라도 있어야 한다
```

## 설정 바꾸고 반영하기

컨테이너를 재시작할 필요가 없습니다. `conf.d/` 는 **디렉터리** 바인드 마운트라 호스트에서 고친 게 바로 보입니다.

> ⚠ `Caddyfile` 자체는 다릅니다. **파일** 마운트라 inode 에 고정돼서, 호스트에서 고쳐도 컨테이너는 옛 내용을 계속 봅니다. `reload` 가 조용히 옛 설정을 다시 로드합니다. 전역 옵션이나 catch-all 을 바꿨다면 `docker compose up -d --force-recreate caddy` 가 필요합니다. 자세한 내용은 [Caddy 함정 모음](caddy-gotchas.ko.md) 문서를 참고하세요.

```sh
# 문법 검사 먼저 — 깨진 설정으로 리로드하면 기존 설정이 유지되지만 확인하는 게 낫다
docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile

docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

> ⚠ 검사는 `caddy adapt` 가 아니라 **`caddy validate`** 로 하세요. `adapt` 는 문법만 보고 통과시키는 오류가 있습니다 — `{args[:]}` 인자 누락이 대표적인데, adapt 는 통과하고 실제 기동에서 죽습니다. `validate` 는 provision 단계까지 돌려서 이걸 잡아냅니다. 자세한 내용은 [Caddy 함정 모음](caddy-gotchas.ko.md) 문서를 참고하세요.

리로드가 실패하면 기존 설정이 그대로 살아 있습니다. 로그를 보세요.

```sh
docker compose logs --tail 50 caddy
```

## OIDC 게이트까지 띄우기

```sh
cp oauth2-proxy/oidc.cfg.example oauth2-proxy/oidc.cfg
vi oauth2-proxy/oidc.cfg          # issuer, client_id, redirect_url
vi .env                            # O2P_CLIENT_SECRET, O2P_COOKIE_SECRET
docker compose --profile oidc up -d
```

`--profile oidc` 없이 `up -d` 하면 oauth2-proxy 는 기동하지 않습니다. 설정 절차는 [OIDC 게이트](oidc.ko.md) 문서를 참고하세요.

## DNS-01 이 필요할 때

80 포트가 공인망에서 도달 불가라 HTTP-01 을 못 쓰는 경우입니다. rfc2136 플러그인이 들어간 이미지를 빌드해야 합니다.

```sh
vi docker-compose.yml     # image: caddy:2 를 주석처리하고 build: ./caddy 를 켠다
vi .env                   # DNS_TSIG_* 채우기
docker compose up -d --build
```

자세한 내용은 [TLS](tls.ko.md) 문서를 참고하세요.

## 업데이트

```sh
cd /opt/xproxy
git pull
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

`conf.d/` 의 자기 사이트 파일과 `.env`, `oauth2-proxy/*.cfg` 는 `.gitignore` 대상이거나 새 파일이라 `git pull` 로 덮이지 않습니다. 다만 `00-*.caddy` 스니펫을 직접 고쳤다면 충돌합니다 — 스니펫은 고치지 말고 사이트 파일에서 인자로 조절하는 쪽을 권합니다.

이미지 자체를 올리려면:

```sh
docker compose pull && docker compose up -d
```

## 로그 위치

```
logs/<hostname>/access.log        사이트별
logs/_catchall/access.log         IP 직접 접근, 미등록 호스트, SNI 미스
```

로그 스키마는 [로그](logging.ko.md) 문서를 참고하세요.

## 제거

```sh
cd /opt/xproxy
docker compose down
```

발급받은 인증서와 ACME 계정키는 named volume 에 있어서 `down` 만으로는 지워지지 않습니다. 완전히 지우려면 `docker compose down -v` 인데, **ACME 계정키가 같이 날아갑니다.** 다시 올릴 계획이라면 볼륨은 남겨두세요 — Let's Encrypt 는 발급 한도가 있습니다.
