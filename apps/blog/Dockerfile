# Static blog image. CI builds the Hugo site into site/public/ first
# (pinned + checksum-verified hugo, see .github/workflows/blog-build.yml),
# then this image just bakes the output into Caddy.
# Base digest-pinned (immutable): caddy:2.11.4-alpine
FROM caddy:2.11.4-alpine@sha256:77c07d5ebfa5be9fd6c820d2094ae662c9e7eeb9bf98346b7f639900263ee2a2

# Upstream sets cap_net_bind_service file capabilities on the binary; we
# listen on :8080 so it's unused, and file caps break exec under the
# Deployment's no-new-privs hardening. Strip them.
RUN apk add --no-cache libcap && setcap -r /usr/bin/caddy && apk del libcap

COPY Caddyfile /etc/caddy/Caddyfile
COPY site/public /usr/share/caddy

EXPOSE 8080
