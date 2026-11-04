FROM caddy:2-builder-alpine AS builder
WORKDIR /build
RUN xcaddy build --with github.com/caddy-dns/cloudflare --with github.com/mholt/caddy-dynamicdns
FROM caddy:2-alpine
COPY --from=builder /build/caddy /usr/local/bin/caddy