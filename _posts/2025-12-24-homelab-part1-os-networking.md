---
layout: default
title: "Homelab Part 1: 2012 Mac mini, Fedora CoreOS and a Modern Home DNS Stack"
date: 2025-12-24
---
# Homelab Part 1: 2012 Mac mini, Fedora CoreOS and a Modern Home DNS Stack

> OS, networking, and DNS/ad‑blocking on a repurposed 2012 Mac mini.

For years I had a **2012 Mac mini** sitting in a drawer. Too old for the latest macOS, but too good to throw away: small, quiet, low power, and perfectly capable of running a few server workloads.

Instead of recycling it, I turned it into a **home infrastructure node**. Today that little machine:

* Runs **Fedora CoreOS**, an immutable, auto‑updating Linux.
* Acts as the **primary DNS server** for the whole house.
* **Blocks ads and trackers** for every device on the network.
* Resolves domains using a **local, validating, recursive resolver** instead of a public DNS service.

On top of that, it also hosts a few other containers (media server, home automation), but this article focuses on the **OS and DNS/ad‑blocking layer**.

I call this machine **`asteria`**.

---

## What this server actually does

At the networking level, there are two key services:

* **Pi-hole** – A DNS “sinkhole” that filters ads and trackers. It becomes the DNS server handed out by the home router, so every device at home benefits automatically.
* **Unbound** – A local, validating, recursive DNS resolver. Instead of forwarding queries to a public resolver, it walks the DNS hierarchy from the root servers down.

The data path looks like this:

```text
Devices at home → Pi-hole (on asteria) → Unbound (on asteria) → DNS root servers
```

Pi-hole decides *what* to block or allow. Unbound does the *actual resolution*.

The rest of the article explains how I set up the OS and wired these two services together in a way that is low‑maintenance and easy to recreate.

---

## Where the Mac mini fits in the home network

The physical network is simple:

```text
Internet → Starlink modem → Deco X20 router (Wi‑Fi + DHCP)
                               → TP-Link SG105 switch → asteria (Mac mini)
                                                         └→ printer
                                                         └→ other wired devices
```

The important pieces are:

* The **Deco X20** is the home router and DHCP server.
* The Mac mini (`asteria`) is plugged into the switch via Ethernet.
* The Deco is configured with a **DHCP reservation** so `asteria` always gets the same IP address.
* All other devices (phones, laptops, tablets, TVs) get their IP and DNS settings from the Deco.

Later, the Deco will hand out `asteria` as the DNS server via DHCP, so all devices use Pi-hole and Unbound without any per‑device changes.

---

## Why Fedora CoreOS for an old Mac mini?

I wanted this machine to behave more like **appliance‑style infrastructure** and less like a traditional Linux box that needs manual patching and cleanup.

Fedora CoreOS (FCOS) fit that goal well:

* It’s designed to run **containers** as its primary workload.
* It applies **automatic, transactional updates**: if an update fails, it rolls back.
* The base system is largely **immutable**; configuration is applied at first boot via Ignition.
* It comes with **Podman** and integrates with systemd through **Quadlet**, so containers can be defined as familiar systemd units.

If the system ever gets into a bad state, I don’t have to debug years of drift. I can reinstall FCOS and reapply the same configuration from version‑controlled files.

---

## Defining the base system with Butane

Fedora CoreOS uses **Ignition** for its first‑boot configuration. Ignition files are JSON, but they’re usually generated from a more readable **Butane** YAML file.

In my Git repo, I keep a file like `butane/asteria.bu` that describes the basics of the system. A simplified excerpt looks like this:

```yaml
variant: fcos
version: 1.5.0
metadata:
  name: asteria
  labels:
    role: homelab

passwd:
  users:
    - name: core
      groups: [ wheel ]
      ssh_authorized_keys:
        - ssh-ed25519 AAAA...your-public-key-here

storage:
  files:
    - path: /etc/hostname
      mode: 0644
      contents:
        inline: asteria

    - path: /etc/systemd/journald.conf.d/asteria.conf
      mode: 0644
      contents:
        inline: |
          [Journal]
          SystemMaxUse=200M
          MaxRetentionSec=2week

  directories:
    - path: /var/srv
      mode: 0755
    - path: /var/srv/containers
      mode: 0755

systemd:
  units:
    - name: systemd-resolved.service
      mask: true
```

This is enough to:

* Create the `core` user with my SSH key.
* Set the hostname to `asteria`.
* Put the system journal on a diet (keep logs to ~200MB or ~2 weeks).
* Create a base `/var/srv` directory for all future service data.
* Mask `systemd-resolved` so it doesn’t bind port 53 (Pi-hole will use that).

The full file in the repo includes more comments, but this gives an idea of the shape.

To generate the Ignition file used by the installer, I run butane on my laptop using Docker. I usually run these one-off commands using Docker so I don't have unused binary leftovers on my computer ;-)

```bash
docker run --rm -i quay.io/coreos/butane:release \
  --pretty --strict < config.bu > config.ign
```

The Ignition file is now ready to be served over HTTP:

```
python3 -m http.server 8000
``` 

---

## Installing Fedora CoreOS on the Mac mini

With the Ignition file ready, the installation was straightforward:

1. Download the Fedora CoreOS ISO image for bare metal (x86_64).

2. Write the ISO to a USB drive.

3. Boot the Mac mini from that USB (holding the Option key at startup and selecting the USB).

4. In the live FCOS environment, install to the internal disk with:

   ```bash
   sudo coreos-installer install /dev/sda  --stream stable  --ignition-url http://192.168.1.50:8000/config.ign
   ```

5. Reboot:

   ```bash
   sudo reboot
   ```

On first boot, Ignition applies everything defined in `asteria.bu`: user, SSH key, hostname, journald limits, base directories, and the masking of `systemd-resolved`.

Once the system comes up and receives its address from the router, I can simply:

```bash
ssh core@asteria
```

and I’m in.

At this point, `asteria` is just a clean Fedora CoreOS machine with some basic opinionated defaults and an empty `/var/srv/containers` waiting for services.

---

## A simple layout for DNS services

I don’t want configuration and state for my services scattered across the filesystem. For sanity and backups, everything lives under `/var/srv/containers`.

For the DNS stack, I use:

```text
/var/srv/containers/
  pihole/
  unbound/
```

Each directory holds configuration and state for its service. For example, Pi-hole keeps its configuration and dnsmasq snippets in the `pihole/` subtree; Unbound stores its configuration in `unbound/`.

On the host, that’s as simple as:

```bash
sudo mkdir -p /var/srv/containers/pihole
sudo mkdir -p /var/srv/containers/unbound
```

Later, the containers will mount these directories.

---

## Defining containers with systemd/Quadlet

Fedora CoreOS includes **Quadlet**, which allows you to define containers using simple `.container` files under `/etc/containers/systemd/`. systemd reads these and generates regular service units.

This means I can manage containers just like any other systemd service:

* `systemctl enable --now pihole.service`
* `systemctl status unbound.service`

A typical Quadlet file looks like this (Pi-hole, shortened):

```ini
[Unit]
Description=Pi-hole DNS + Web UI
Wants=network-online.target
After=network-online.target

[Container]
ContainerName=pihole
Image=docker.io/pihole/pihole:latest
Network=host

Volume=/var/srv/containers/pihole:/etc/pihole:z
EnvironmentFile=/var/srv/containers/pihole/pihole.env

[Service]
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Once this file exists as `/etc/containers/systemd/pihole.container`, I run:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now pihole.service
```

and Pi-hole becomes a normal systemd service.

I use the same pattern for Unbound: one directory under `/var/srv/containers`, one `.container` file under `/etc/containers/systemd/`.

---

## Pi-hole: the visible part of the DNS stack

Pi-hole is the component the rest of the family indirectly “feels”. It:

* Accepts DNS queries from all devices at home.
* Applies one or more blocklists to filter out ads and trackers.
* Exposes a web UI with useful statistics and controls.

The basic Pi-hole environment for this setup is very small. In `/var/srv/containers/pihole/pihole.env` I keep, for example:

```bash
TZ=America/Argentina/Cordoba
WEBPASSWORD=change_me
PIHOLE_DNS_=127.0.0.1#5335
REV_SERVER=false
```

The important line is `PIHOLE_DNS_`: it tells Pi-hole to forward allowed queries to Unbound, which listens on localhost port 5335 on the host.

Once the service is up, the Pi-hole dashboard is reachable from any browser on the LAN. From there I can:

* Configure blocklists.
* Check which domains are being requested and blocked.
* Temporarily disable blocking if needed.

On a typical day in my house, the dashboard shows:

* **Total DNS queries**: tens of thousands.
* **Blocked queries**: often in the 10–20% range.

The exact numbers depend on which devices are active, but the ratio makes it very clear how much noise on the network is ads, tracking, and background telemetry.

---

## Unbound: local, validating, recursive DNS

Pi-hole needs an upstream resolver. Instead of using a public DNS service, I run **Unbound** locally as a validating, recursive resolver.

The Unbound configuration file lives under `/var/srv/containers/unbound`, and the container is defined with a Quadlet unit very similar to Pi-hole’s. The key differences are:

* It uses the `docker.io/mvance/unbound:latest` image.
* It publishes port 53 only on localhost, mapped to a non‑conflicting port on the host (I use 5335).

In the Quadlet file, that looks like:

```ini
[Container]
ContainerName=unbound
Image=docker.io/mvance/unbound:latest

PublishPort=127.0.0.1:5335:53/udp
PublishPort=127.0.0.1:5335:53/tcp

Volume=/var/srv/containers/unbound/unbound.conf:/opt/unbound/etc/unbound/unbound.conf:z
Environment=TZ=America/Argentina/Cordoba
```

The Unbound configuration itself enables recursion and basic hardening, and relies on the built‑in root hints so it can walk the DNS hierarchy directly.

One of the advantages of running Unbound locally is that I can test it in isolation from Pi-hole. From the host:

```bash
nslookup -port=5335 example.com 127.0.0.1
```

If that returns a valid answer, I know the recursive resolver is working and any issues are likely in the layer above (Pi-hole), not in DNS resolution itself.

---

## Wiring Pi-hole and Unbound together

With both services running on `asteria`, the DNS flow is:

```text
Devices → Pi-hole → Unbound → DNS root servers
```

On the Pi-hole side, in the web UI:

1. I open **Settings → DNS**.
2. I make sure the only upstream is `127.0.0.1#5335`.
3. I disable any public resolvers that Pi-hole offers by default.

On the Unbound side, I keep its container bound only to localhost, so nothing else on the network can query it directly. All DNS traffic comes through Pi-hole first.

This separation of roles keeps the configuration easy to reason about:

* **Pi-hole** decides *whether* to answer a query, block it, or rewrite it.
* **Unbound** decides *how* to resolve it, by consulting the DNS hierarchy.

---

## Making it transparent via the router

The last step is to make all this invisible to everyone else in the house.

On the Deco router, I change the **LAN DNS setting** so that DHCP clients receive `asteria` as their DNS server. After devices renew their leases or reconnect to Wi‑Fi, they automatically start sending DNS queries to Pi-hole.

From that point on:

* There is no need to touch individual devices.
* New devices joining the network immediately benefit from ad‑blocking and the local resolver.
* The Pi-hole dashboard becomes a nice, passive view into the network’s DNS traffic.

If the Mac mini is ever down for maintenance, I can temporarily point the router’s DNS back to an external resolver and restore it later.

---

## Lessons learned

A few takeaways from this project:

* **Old hardware has plenty of life left.** A 2012 Mac mini handles a modern Linux, Pi-hole, and Unbound without effort. For home infrastructure, CPU is rarely the limiting factor.
* **Immutable OS + containers reduce long‑term friction.** Fedora CoreOS with Quadlet means most behavior is defined by a small set of files. That’s easier to understand and reproduce than years of incremental tweaks.
* **A clear layout pays for itself.** Keeping service state together under `/var/srv/containers` makes documentation and future changes much simpler.
* **Splitting filtering and resolution is clean.** Let Pi-hole specialize in policy (block/allow) and Unbound specialize in resolution. Each component does one thing well.
* **The router is the right integration point.** Changing DNS at the DHCP source keeps things transparent for the rest of the family and for future devices.
* **Good diagnostics save time.** Being able to test Unbound directly, and to see detailed DNS stats in Pi-hole, made it much easier to debug early issues like port conflicts or access control.

The end result is a small, quiet box doing an important job 24/7: improving privacy and cutting down a surprising amount of unwanted traffic for every device on the network.
