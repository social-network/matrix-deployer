# Setup Matrix Server

This guide can be used to setup a Matrix x86 server running CentOS (7 only for now; 8 is not yet supported), Debian (9/Stretch+), Ubuntu (16.04+), or Archlinux. This playbook doesn't support running on ARM (see), however a minimal subset of the tools can be built on the host, which may result in a working configuration, even on a Raspberry pi (see Alternative Architectures). We only strive to support released stable versions of distributions, not betas or pre-releases. This playbook can take over your whole server or co-exist with other services that you have there.

## Ubuntu Guide

To begin ssh as root into your machine and running

```
sudo apt update
sudo apt install software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
sudo apt install python
sudo apt install python-pip
sudo apt install cron
pip install dnspython
```

### DNS Setup

To set up Matrix on your domain, you'd need to do some DNS configuration.

To use an identifier like @<username>:<your-domain>, you need to instruct the Matrix network that Matrix services for <your-domain> are delegated over to matrix.<your-domain>. As we discuss in Server Delegation, there are 2 different ways to set up such delegation:

Option 1) Serving a https://<your-domain>/.well-known/matrix/server file (from the base domain!)
Option 2) using a \_matrix.\_tcp DNS SRV record (don't confuse this with the \_matrix-identity.\_tcp SRV record described below)

This playbook mostly discusses the well-known file method, because it's easier to manage with regard to certificates. If you decide to go with the alternative method (Server Delegation via a DNS SRV record (advanced)), please be aware that the general flow that this playbook guides you through may not match what you need to do.

### General outline of DNS settings you need to do

| Type  | Host                         | Priority | Weight | Port | Target                 |
| ----- | ---------------------------- | -------- | ------ | ---- | ---------------------- |
| A     | `matrix`                     | -        | -      | -    | `matrix-server-IP`     |
| CNAME | `riot`                       | -        | -      | -    | `matrix.<your-domain>` |
| CNAME | `dimension` (*)              | -        | -      | -    | `matrix.<your-domain>` |
| CNAME | `jitsi` (*)                  | -        | -      | -    | `matrix.<your-domain>` |
| SRV   | `_matrix-identity._tcp`      | 10       | 0      | 443  | `matrix.<your-domain>` |

DNS records marked with `(*)` above are optional. They refer to services that will not be installed by default (see the section below). If you won't be installing these services, feel free to skip creating these DNS records.

### Subdomains setup

As the table above illustrates, you need to create 2 subdomains (`matrix.<your-domain>` and `riot.<your-domain>`) and point both of them to your new server's IP address (DNS `A` record or `CNAME` record is fine).

The `riot.<your-domain>` subdomain is necessary, because this playbook installs the Riot web client for you.

If you'd rather instruct the playbook not to install Riot (`matrix_riot_web_enabled: false` when configuring the playbook, feel free to skip the `riot.<your-domain>` DNS record.

The `dimension.<your-domain>` subdomain may be necessary, because this playbook could install the [Dimension integrations manager](http://dimension.t2bot.io/) for you. Dimension installation is disabled by default, because it's only possible to install it after the other Matrix services are working (see [Setting up Dimension](configuring-playbook-dimension.md) later). If you do not wish to set up Dimension, feel free to skip the `dimension.<your-domain>` DNS record.

The `jitsi.<your-domain>` subdomain may be necessary, because this playbook could install the [Jitsi video-conferencing platform](https://jitsi.org/) for you. Jitsi installation is disabled by default, because it may be heavy and is not a core required component. To learn how to install it, see our [Jitsi](configuring-playbook-jitsi.md) guide. If you do not wish to set up Jitsi, feel free to skip the `jitsi.<your-domain>` DNS record.

### `_matrix-identity._tcp` SRV record setup

To make the [ma1sd](https://github.com/ma1uta/ma1sd) Identity Server (which this playbook installs for you) be authoritative for your domain name, set up one more SRV record that looks like this:
- Name: `_matrix-identity._tcp` (use this text as-is)
- Content: `10 0 443 matrix.<your-domain>` (replace `<your-domain>` with your own)

### Network settings

If your server is running behind another firewall, you need to open these ports:

- 80/tcp (HTTP webserver)
- 443/tcp (HTTPS webserver)
- 3478/tcp (TURN over TCP)
- 3478/udp (TURN over UDP)
- 5349/tcp (TURN over TCP)
- 5349/udp (TURN over UDP)
- 8448/tcp (Matrix Federation API HTTPS webserver)
- the range 49152-49172/udp (TURN over UDP)
- 4443/tcp (Jitsi Harvester fallback)
- 10000/udp (Jitsi video RTP)
