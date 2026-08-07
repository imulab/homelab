# Custom Caddy image bundling:
#   - caddy-dns/cloudflare        : Cloudflare DNS-01 for wildcard certs
#   - lucaslorentz/caddy-docker-proxy : label-based backend discovery
#
# The official caddy image includes neither; caddy-docker-proxy's prebuilt
# images include the proxy module but not the Cloudflare DNS plugin, so we
# build our own once with xcaddy.
FROM caddy:2-builder AS builder
RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare \
    --with github.com/lucaslorentz/caddy-docker-proxy/v2

FROM caddy:2-alpine
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
# Required to start the caddy-docker-proxy controller alongside the Caddy
# server. Without this, the base image runs `caddy run` and the label-discovery
# module never activates — Caddy serves nothing and watches no containers.
CMD ["caddy", "docker-proxy"]
