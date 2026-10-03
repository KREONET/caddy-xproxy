# 로그

[🇺🇸 English](logging.md) · [🇰🇷 한국어](logging.ko.md)

## 위치

```
logs/<hostname>/access.log        사이트별
logs/_catchall/access.log         IP 직접 접근 · 미등록 호스트 · SNI 미스
```

50MiB 마다 롤, 14개 + 90일 보관. 롤된 파일은 `access-<타임스탬프>-size.log.gz` 로 남습니다.

사이트마다 파일을 나누는 이유는 한 사이트의 스캐너 폭주가 다른 사이트 로그를 밀어내지 않게 하기 위해서입니다.

## _catchall 이 중요한 이유

`conf.d/` 의 어떤 사이트에도 매칭되지 않은 접근이 여기 쌓입니다. IP 로 직접 들어온 요청, 등록되지 않은 호스트명, SNI 가 맞지 않는 접속이 모두 여기 해당합니다.

**정상 사용자는 여기 올 이유가 없습니다.** 그래서 이 파일은 "누가 우리 IP 대역을 훑고 있나"의 가장 깨끗한 소스입니다.

## 스키마

Caddy 기본 JSON 로그에 이 레포가 필드를 덧붙인 형태입니다.

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

`client_ip` 를 쓰세요. `remote_ip` 도 있지만 프록시 체인이 끼면 달라집니다.

## 덧붙는 필드

| 필드 | 값 | 붙는 곳 |
|--|--|--|
| `block_reason` | `probe-secret` · `probe-app` · `probe-ext` | 스캐너 3레이어 |
| | `ai-training-crawler` · `empty-user-agent` · `ai-assistant-fetch` | badbots |
| | `geo-denied` | geo-allow |
| `would_block` | `probe-app` · `probe-ext` | 관찰 모드 (차단하지 않음). 앞 레이어가 차단한 요청에도 같이 붙는다 |
| `auth` | `oidc` · `internal-bypass` | OIDC 게이트 사이트 |
| | `gate-stub` | OIDC 를 완주 못하는 클라이언트를 종결시킨 요청 |
| `auth_sub` | OIDC sub 클레임 | `auth=oidc` 일 때 |
| `auth_email` | 이메일 | `auth=oidc` 일 때 |

## status 읽는 법

| status | 뜻 |
|--|--|
| `444` | 스캐너 3레이어 또는 `geo-allow` 가 차단. 빈 본문 |
| `403` | badbots 가 차단 |
| `0` | 연결이 끊김 — `abort`. 내부전용 사이트의 외부 접근, catch-all |
| 그 외 | 백엔드가 응답한 것 |

`status: 0` 에는 `block_reason` 이 **없습니다.** `abort` 가 `log_append` 필드를 날리기 때문입니다. 그래서 사유를 남겨야 하는 차단은 전부 `respond` 로 처리합니다. 근거는 [Caddy 함정 모음](caddy-gotchas.ko.md) 문서에 정리해 두었습니다.

## 자주 쓰는 질의

**차단 사유별 집계**

```sh
jq -r 'select(.block_reason) | .block_reason' logs/*/access.log \
  | sort | uniq -c | sort -rn
```

**가장 시끄러운 스캐너 IP**

```sh
jq -r 'select(.block_reason) | .request.client_ip' logs/*/access.log \
  | sort | uniq -c | sort -rn | head -20
```

**관찰 모드에서 걸린 것 — 여기 정상 트래픽이 섞여 있으면 빠뜨린 네임스페이스다**

```sh
jq -r 'select(.would_block) | select(.block_reason | not)
         | "\(.would_block)\t\(.request.uri)"' logs/*/access.log \
  | sort | uniq -c | sort -rn | head -50
```

**특정 사용자가 뭘 했나 (OIDC 게이트 사이트)**

```sh
jq -r 'select(.auth_email=="alice@example.com")
       | "\(.ts)\t\(.status)\t\(.request.uri)"' logs/app.example.com/access.log
```

`auth_email` 은 바뀔 수 있습니다. 장기 추적이 목적이면 `auth_sub` 를 기준키로 쓰세요 — OIDC sub 클레임은 불변입니다.

**롤된 로그까지 포함해서**

```sh
zcat -f logs/*/access*.log* | jq -r 'select(.block_reason) | .request.client_ip' \
  | sort | uniq -c | sort -rn | head
```

## fail2ban 연계

이 레포는 탐지와 로깅까지 담당하고, 밴은 별도입니다. 필요한 건 다 로그에 있습니다.

```ini
# 대략 이런 모양이 됩니다
[Definition]
failregex = ^.*"client_ip":"<HOST>".*"block_reason":"probe-(secret|app|ext)".*$
```

몇 가지 주의점.

**`status` 로 매칭하지 마세요.** `abort` 로 끊긴 요청은 `status: 0` 이고 사유가 없습니다. `block_reason` 이 유일하게 믿을 수 있는 키입니다.

**`ignoreip` 에 내부 대역을 넣으세요.** 취약점 점검 스캐너를 사내에서 돌리면 자기 자신을 밴합니다.

**로그 롤링을 고려하세요.** Caddy 는 rename + 새 파일 방식으로 롤합니다. fail2ban 은 inode 변경을 따라가지만, `logpath` 글롭은 `access.log` 만 잡도록 두세요. 롤된 `.gz` 까지 잡으면 과거 기록으로 재밴이 일어납니다.

**1-strike 로 갈 거면 관찰 모드를 먼저 돌리세요.** `probe-*` 는 원리상 오탐이 없어야 하지만, 업로드 네임스페이스를 하나 빠뜨리면 실사용자가 잘려나갑니다. 도입 절차는 [스캐너 차단 설계](scanner-defense.ko.md) 문서를 참고하세요.

## 로그가 안 보일 때

호스트의 `logs/` 디렉터리 권한을 확인하세요. 컨테이너의 Caddy 가 쓸 수 있어야 합니다. 사이트별 하위 디렉터리는 첫 요청이 올 때 자동 생성됩니다 — 사이트를 추가하고 아직 아무도 안 왔으면 디렉터리도 없습니다.
