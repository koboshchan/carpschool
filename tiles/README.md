# Production North America tiles

The production compose service mounts the existing dataset at
D:/Carpool/openmaptilesdata read-only. It does not copy the 16 GB MBTiles file
or alter the legacy D:/Carpool stack. Data, Noto fonts and built style/sprites
remain in that directory. The config here exposes the old style as both
positron (the web app's existing route) and OSM OpenMapTiles.

The web app retains TILESERVER_URL=http://tiles:8080 and its /tiles proxy.
The vector source is openmaptiles; overlay sources route/circles are independent.

In E:/oneserver/settings.json, configure an HTTPS entry for
openmaptiles.services.carpschool.ca forwarding to carpschool-prod-tiles-1:8080,
with ca-bundle=fullchain.pem and private-key=privkey.pem. Oneserver owns
carpschool_bridge; the tiles service joins that external network without
publishing a host port. Run the existing oneservercertbot renewal/rebuild
workflow after adding the domain. Do not edit settings.json.example.

The tiles service builds the owned Dockerfile in this directory instead of
pulling a floating upstream latest image. It preserves the pinned upstream
TileServer GL 5.6.0 application, native dependencies and entrypoint, replacing
only its Node 24 binary with the patched 24.22.1 glibc binary. Both node and
nodejs must report that exact version at build time. The upstream Ubuntu and
official bookworm Node images use compatible glibc; Node remains on the same
major and native addon ABI. Do not substitute an Alpine/musl binary here.

Build with docker compose build tiles before an authorized release. Verify
node --version, /health, vector metadata and a raster tile against the mounted
dataset after release. A source push does not update the running container.
