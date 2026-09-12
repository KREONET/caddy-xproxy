# Caddy with a DNS plugin, needed only for DNS-01. Enable build: ./caddy in compose.
# DNS-01 이 필요할 때만 쓰는 플러그인 포함 이미지. compose 의 build: ./caddy 를 켠다.
# refer to docs/tls.md
FROM caddy:2-builder AS builder
RUN xcaddy build \
	--with github.com/caddy-dns/rfc2136

FROM caddy:2
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
