---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\res\xml\network_security_config.xml
---

# app\src\main\res\xml\network_security_config.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<!--
  Cleartext is permitted, and it is worth writing down why rather than leaving a reader to
  assume nobody thought about it.

  Her face socket speaks plain HTTP and ws. It binds loopback by default and, when remote
  access is switched on, is meant to be reached over the UDM SE's Wireguard - where the
  traffic is encrypted on the wire regardless of what this app negotiates. Her socket is
  never forwarded; the only forwarded port is Wireguard's own.

  This is not scoped to a domain list because there is nothing to list: the host is typed in
  by the person using the app and is whatever her LAN address happens to be. A config that
  named 10.1.1.x would be a fiction the moment the subnet changed.

  The honest cost: on an untrusted network without the VPN up, this app would send a typed
  message in the clear. That is the same exposure her own socket already has and is the
  reason remote access is off by default. TLS on her listener is the real fix and belongs in
  her repo, not in a workaround here - at which point this file gets tightened.
-->
<network-security-config>
    <base-config cleartextTrafficPermitted="true">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
</network-security-config>
```
