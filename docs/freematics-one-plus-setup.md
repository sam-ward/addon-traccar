# Freematics ONE+ Traccar Edition — Setup Notes

## Overview

The Freematics ONE+ is an OBD-II dongle with an ESP32 microcontroller, GPS, and
cellular connectivity. The "Traccar Edition" ships with the
[telelogger firmware](https://github.com/stanleyhuangyc/Freematics/tree/master/firmware_v5/telelogger)
which is open source.

---

## How the dongle communicates

The firmware supports three protocols, configured in `config.h`:

```c
#define PROTOCOL_UDP 1         // default
#define PROTOCOL_HTTPS_GET 2
#define PROTOCOL_HTTPS_POST 3
```

**Default behaviour**: UDP binary packets to `hub.freematics.com:8081` using a
proprietary Freematics format. This goes to the Freematics cloud relay — it does
NOT connect directly to Traccar out of the box.

**The HTTPS options** also target the Freematics Hub (`hub.freematics.com/hub/api`
on port 443) — not OsmAnd, despite what the Traccar addon docs imply.

Traccar has **native support for the Freematics UDP protocol** via `freematics.port`
(default 5170). So the Freematics Hub can be cut out entirely by pointing the
firmware straight at a self-hosted Traccar instance.

---

## Network path (cellular → HA)

```
Car dongle (OBD-II port)
    │  UDP or HTTPS, cellular data
    ▼
Mobile carrier network
    │
    ▼
Public internet
    │
    ▼
Home router  ◄── port-forward needed (or Cloudflare Tunnel, see below)
    │
    ▼
HA host  (host_network: true, so Traccar ports are on the host directly)
    │
    ▼
Traccar process on port 5170 (Freematics protocol)
```

The addon uses `host_network: true` in `config.yaml`, which means Traccar's ports
are bound directly to the HA host's network interface — no Docker NAT in the way.

---

## Option A: Router port-forward (simplest)

1. Forward **UDP port 5170** on the router to the HA host's LAN IP.
2. Set up a DDNS hostname pointing at your home IP (e.g. DuckDNS).
3. Flash firmware with updated `config.h` (see below).
4. Enable `freematics.port` in `traccar.xml` (see below).

---

## Option B: Cloudflare Tunnel (no port-forward)

Because the HTTPS protocol option already uses TLS, it can be routed through a
Cloudflare Tunnel without any router changes:

1. Install the [Cloudflare Tunnel HA addon](https://github.com/brenner-tobias/addon-cloudflared).
2. Expose Traccar's port via a public `https://traccar.yourdomain.com` hostname.
3. Configure the firmware to use `PROTOCOL_HTTPS_GET` or `PROTOCOL_HTTPS_POST`
   pointing at that hostname.

> **Not yet investigated** — the Freematics HTTPS payload format (`/hub/api`) may
> need to be adapted to match what Traccar's HTTP-based endpoints expect. Needs
> testing.

---

## Firmware changes required (`config.h`)

```c
// Point directly at your Traccar instance instead of hub.freematics.com
#define SERVER_HOST "your-ddns-hostname.duckdns.org"
#define SERVER_PROTOCOL PROTOCOL_UDP
#define SERVER_PORT 5170

// Set your cellular APN
#define CELL_APN "your.carrier.apn"
```

Build and flash via PlatformIO (`platformio.ini` is in the firmware directory).

---

## Addon changes required

### 1. Enable the Freematics port in `traccar/rootfs/etc/traccar/traccar.xml`

Uncomment the following line (it's in the examples section):

```xml
<entry key='freematics.port'>5170</entry>
```

### 2. Expose the port in `traccar/config.yaml`

Add the UDP port alongside the existing web UI port:

```yaml
ports:
  80/tcp: 8082
  5170/udp: 5170
ports_description:
  80/tcp: Web interface
  5170/udp: Freematics device protocol
```

Then rebuild and reinstall the local addon, and add the port-forward on the router.

---

## Device identification in Traccar

The dongle identifies itself to Traccar using a device ID derived from its hardware
(likely the ESP32 chip ID or a configured value). When the dongle first connects,
Traccar will log an unknown device. You then register that ID in the Traccar web UI
under **Devices → Add** and paste in the ID from the Traccar logs.

---

## TODO / not yet done

- [ ] Enable `freematics.port` in `traccar.xml` and `config.yaml`
- [ ] Test UDP connectivity end-to-end once router port-forward is configured
- [ ] Investigate whether Cloudflare Tunnel + HTTPS protocol option is viable
- [ ] Confirm exact device ID format the firmware sends (check Traccar logs on
      first connection)
