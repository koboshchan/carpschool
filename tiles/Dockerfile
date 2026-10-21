# Match the upstream Ubuntu image's glibc and Node 24 native-addon ABI.
FROM node:24.22.1-bookworm-slim AS patched-node

# Keep the tested TileServer GL app, native renderers and entrypoint unchanged.
FROM maptiler/tileserver-gl@sha256:3a9ccdb24820b6814c8119bcc8a4376c39867cb0ffe69d62919ef898b90c2427
USER root
COPY --from=patched-node /usr/local/bin/node /usr/bin/node
RUN test "$(node --version)" = "v24.22.1" \
    && test "$(nodejs --version)" = "v24.22.1"
USER node:node
