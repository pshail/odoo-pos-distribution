# Odoo Point of Sale - PoS - Distribution

❗**ARCHIVED** ❗

This project was based on Odoo 12.
When I started to work on it again, Odoo 15 was the current version.
The Point of Sale application has _greatly_ improved since Odoo 12:

* Receipt printing on network printers fully available out-of-the box
* No iotbox needed anymore
* Many quirks and workarounds present in this repo are outdated and not necessary anymore

I'm now running Odoo in the cloud, no local deployment on a Raspberry Pi needed anymore.

For this reason, I'm archiving this project and won't update any further.

---

Dockerfiles, Docker Compose and Kubernetes configuration for running an Odoo based PoS
in containers.

## Quickstart

### Docker Compose

1. `docker-compose up`

Continue with "Odoo PoS configuration"

### Kubernetes / K3s

The provided YAML files have been developed and tested with [k3s](https://k3s.io/) on amd64 and arm64.
Installation will happen in the namespace `pos`.

1. Install [k3s](https://k3s.io/), f.e. using [k3sup](https://k3sup.dev/) (any other Kubernetes should work as well)
1. Install [local-path-provisioner](https://github.com/rancher/local-path-provisioner)
1. Apply the deployment YAMLs: `kubectl apply -f deployment/`

Continue with "Odoo PoS configuration"

## Odoo PoS configuration

1. Connect to Odoo and create a new database
   On Kubernetes the Ingress defines the hostnames `pos` and `iotbox`
1. Install some Odoo Addons:
  * "Point of Sale"
  * "POS Network Printer"
1. (Enable "Developer mode" under Odoo settings)
1. Configure PoS for IoT Box (see screenshots under `docs/`)
1. Configure Order Printer (see screenshots under `docs/`)

## Odoo Addons

The default addons from Odoo core are not enough for a smooth PoS experience,
therefore a good amount of PoS addons can be used. This distributions adds
the following addon sources:

* [pos-addons](https://github.com/it-projects-llc/pos-addons):
  * `hw_printer_network` && `pos_printer_network`: Support for network receipt printer
* [odoo-cloud-platform](https://github.com/camptocamp/odoo-cloud-platform):
  * `monitoring_status`: Status endpoint for Odoo
* [CybroAddons](https://github.com/CybroOdoo/CybroAddons):
  * `pos_product_category_filter`: Only show selected product categories on the PoS view

The following addons are delivered in this repository:

* `ip_pos_ticket_order_number`: Custom made module to print a big order number both on
  the receipt and kitchen order
* `pos_product_sequence`: Commercial module to manually order products (default is
  alphabetically). You're not allowed to use this module unless you bought it as
  well. See [POS Product Sequence](https://apps.odoo.com/apps/modules/12.0/pos_product_sequence/)
  on the Odoo app store.

## Hardware and Networking

```
                                                    +
                           +----------------+       |
                           | Raspberry Pi 4 |   WLAN|
                           | (K3s, Pi-hole) | +-----+
                           | DHCP, DNS      |
                           | 192.168.233.9  |
                           +-------+--------+
                                   |
                                   |
                   +---------------+-----------------+
     +---------+   | Router, Switch and Access Point |
     |WLAN         | (Mikrotik RB2011)               |
     |             | 102.168.233.1                   |
     |             +-----+---------------------+-----+
     +                   |                     |
                         |                     |
+-----------+   +--------+-------+     +-------+--------+
| Cash desk |   | Printer 1      |     | Printer 2      |
| tablet    |   | (Cash desk)    |     | (Kitchen)      |
|           |   | 192.168.233.3  |     | 192.168.233.5  |
+-----------+   +-------+--------+     +----------------+
                        |
                        |
                +-------+--------+
                |  Cash drawer   |
                +----------------+
```

* Default network - the posnet - assumed: `192.168.233.0/24`
* IPs:
  * `192.168.233.1`: Mikrotik: WLAN and Switch
  * `192.168.233.3`: Cashdesk receipt printer with cash drawer
  * `192.168.233.5`: Kitchen printer
  * `192.168.233.9`: Raspberry Pi (pospi)
  * `192.168.233.10-50`: DHCP Range
* Printers: [Epson TM-T20II](https://www.epson.ch/products/sd/pos-printer/epson-tm-t20ii)

The Raspberry Pi 4 provides DNS and DHCP to the network with [Pi-hole](https://pi-hole.net/).
Static DNS entries `pos` and `iotbox` are added to `/etc/hosts` which
then are served by the Pi-hole DNS server. The Pi-hole Lighttpd must
be configured to listen on f.e. port 8080 so it doesn't conflict with
K3s ingress. The Raspberry Pi is connected to the PoS network using an
Ethernet connection and to the internet using WLAN. This optionally allows
to use it as an internet gateway by adding some IPtables rules:

```
iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
```

A good thing is to install Wireguard and use it as remote access to the
Raspberry Pi.

When using an Android tablet, a "kiosk" browsing app is the recommended way to
provide the PoS webapp to the user. F.e. [Fully Single App Kiosk](https://play.google.com/store/apps/details?id=com.fullykiosk.singleapp&hl=en).
For iPad just add the webapp to the home screen using Safari, no additional
app is needed.

An UPS is also recommended to power the setup so that short power outages don't
interrupt the ordering process.

## Monitoring

There are two kinds of monitoring prepared: The cluster itself and the PoS
application.

### Simple Cluster Healthcheck

Under `contrib/healthchecks-cronjob.yaml` a simple Kubernetes cronjob is
provided which regularly pings [Healthchecks.io](https://healthchecks.io/).
This helps to see if the whole thing is up and running.

A secret with the ping URL needs to be added before the CronJobs can do it's work:

```
kubectl -n posmon create secret generic healthchecks-io --from-literal=HCURL=https://hc-ping.com/MYUUID
```

### Application and network monitoring

Application monitoring is done using Prometheus, Alertmanager and
Blackbox exporter. No application specific exporters are used, so
it's just a base monitoring to answer the question: "Is it up?".

1. Install [prometheus-operator](https://github.com/coreos/prometheus-operator)
   F.e.: `kubectl apply -f https://raw.githubusercontent.com/coreos/prometheus-operator/master/bundle.yaml`
2. Apply manifests: `kubectl apply -f contrib/posmon/`
3. Create secret for extra scrape config:
   `kubectl -n posmon create secret generic additional-scrape-configs --from-file=contrib/pos-blackbox-exporter-scrape.yaml`
4. Create secret for Alertmanager config:
   `kubectl -n posmon create secret generic alertmanager-posmon --from-file=contrib/alertmanager.yaml`

## Backup

Backup is done using [K8up](https://k8up.io/).

1. Install K8up
2. Apply manifests under `contrib/backup`

### Restore

Restore from Restic and then do a database import:

```
createdb -T template0 restoretest
pg_restore -d restoretest /data/odoo_data.dump
```

## Docker Images

Docker images are automatically built on [Docker Hub](https://cloud.docker.com/repository/docker/tobru/odoo-pos) (for amd64 arch).

* `docker.io/tobru/odoo-pos:latest-iotbox`: IoT Box
* `docker.io/tobru/odoo-pos:latest-pos`: Odoo

## A word on ARM / Raspberry Pi support

TL;DR: It's not easy to run things out of the box on Raspberry Pi / ARM.

Images for ARM64 (f.e. Raspberry Pi) are _not_ automatically built as this
is not supported by Docker Hub. They are built manually on a Raspberry Pi
and uploaded to Docker Hub.

* `docker.io/tobru/odoo-pos:latest-iotbox-arm64v7`: IoT Box
* `docker.io/tobru/odoo-pos:latest-pos-arm64v7`: Odoo

As the [upstream Odoo](https://hub.docker.com/_/odoo/) doesn't support
`linux/arm/v7` even the base image needs to be built on the Raspberry Pi:

1. Clone https://github.com/odoo/docker
2. Change `wkhtmltox` to install `raspbian.stretch_armhf.deb`
3. Build with `docker build -t local/odoo:12 .`
4. Patch local Dockerfiles to use this as base image

A good amount of upstream stuff doesn't work on Raspberry Pi as no multiarch
images are provided. F.e. the proposed monitoring stack with Prometheus doesn't
work out of the box and K8up doesn't provide arm binaries (yet).

The Postgres client installed in the Odoo images is version 9.6 (it's based
on Debian Stretch and upstream doesn't provide `armhf` packages).
If you're using a newer Postgres version, the DB management functionality of
Odoo (Backup/Restore) won't work because of version mismatch.

## Random notes

* Connection from PoS tablet to IoT Box is a direct connection, not via Odoo server!
* Support for opening the cashbox via network printer has been patched. The IP is hardcoded
  to 192.168.233.3. See [0c6ecfdd](https://github.com/tobru/posbox-docker/commit/0c6ecfdd470dad07b9f9c26ecc0fd413c6d605b1)
  and [#730](https://github.com/it-projects-llc/pos-addons/issues/730).
* Odoo 13 will probably change a lot for PoS and will need some additional work. See f.e.
  * Commit: [pos: Replace hw_escpos by PrinterDriver](https://github.com/odoo/odoo/commit/03b2f7b77dc6189bb485e0b834dba5f6d3d4da2c#diff-14c0ea7767d884151475e057ad05ec56)
  * Addon: [pos_epson_printer](https://github.com/odoo/odoo/tree/master/addons/pos_epson_printer)

## TODOs

There are some things which could be improved:

### Distribution

* [ ] Pre-install `monitoring_status` and use for K8s probes
* [ ] Point Blackbox Monitoring to `/monitoring/status`
* [ ] Tweak monitoring rules
* [ ] Mirror important add-ons to this repository
* [ ] Configure `server_wide_modules` (instead of using command line parameters)
  * [ ] Odoo: `base,web,monitoring_status`
  * [ ] Automatically install PoS modules
* [ ] Improve arm builds and overall support (Monitoring, Backup)
* [ ] Support for adding third-party commercial Odoo addons

### PoS Usage

* [ ] Configure default payment option (cash)
* [ ] Don't open cash drawer for virtual payment (Twint, SumUp)
* [ ] Possibility to print a kitchen order ticket per position (not summarized)

## Disclaimer

This is a hobby project and is not actively maintained. I don't provide _any_
support! If you feel like contributing something, that's of course appreciated.
Feel free to open a Pull Request.
