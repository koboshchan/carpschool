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
