+++
title = 'WireGuard in Userspace'
date = 2026-01-15T13:36:09-05:00
draft = false
+++
## WireGuard Continued

In the [previous article](https://manderson-it.ca/posts/wireguard-fun/), we set up a VPN tunnel with the WireGuard kernel module between two sites.

Here, we will create:

- the VPN server and let is act as a DNS resolver for our VPN client,
- generate a QR code for our client configuration, and
- scan the QR code on an iPhone to establish the VPN connection.

{{< rawhtml >}}
<!--
mermaid.js
```js
---
config:
  themeVariables:
    fontSize: 20px
markdownAutoWrap: true
---
graph LR;
    subgraph "fa:fa-house elsewhere"
      IPHONE([fa:fa-phone iPhone])
      QR-\->|fa:fa-camera|IPHONE
      QR([fa:fa-qrcode QR code])
    end
    subgraph "fab:fa-google GCP"
      WG([fab:fa-fort-awesome-alt WireGuard server])
    end
    IPHONE -\-\->|fas:fa-cloud tunnel| WG

graph LR;
    subgraph "fa:fa-house elsewhere"
      IPHONE([fa:fa-phone iPhone])
      QR-\->|fa:fa-camera|IPHONE
      QR([fa:fa-qrcode QR code])
    end
    subgraph "fab:fa-google GCP"
      WG([fab:fa-fort-awesome-alt WireGuard server])
      ROUTER([fa:fa-route Router])
      NAT([fa:fa-network-wired NAT])
      WG-\->ROUTER
      ROUTER-\->NAT
    end
    IPHONE -\-\->|fas:fa-cloud tunnel| WG
    INET([fas:fa-cloud Internet])
    NAT -\-\-> INET
-->
{{< /rawhtml >}}

[![Architecture of the Hamburg and Berlin sites in GCP](/images/wireguard-go-arch.png)](/images/wireguard-go-arch.png)

Last time, we used the WireGuard kernel module. Let's use a userspace implementation.
Why would one want to use a userspace implementation though?

Generally, we want to use WireGuard as a kernel module for best raw performance.
The kernel module will be available in a lot of cases.
However, you may find yourself working with a system that doesn't offer kernel module support.
This could apply to: older or locked-down OSes, some BSD variants, minimal containers or unikernels, or embedded systems.
Similarly, you may need WireGuard as a library if you want to embed it inside an application, you need programmatic control over tunnels, or build a custom vpn client.
In other words, if you are working in a restricted environment, or flexibility and portability are more important, running WireGuard as a userspace process can be a good choice.

Let us choose the well established Golang implementation, called [wireguard-go](https://git.zx2c4.com/wireguard-go).
If you are interested in the conventions for WireGuard userspace implementations,
[take a look here](https://www.wireguard.com/xplatform/).

## Our Environment

:warning: It is for demonstration purposes only.

You need a Google Compute Engine (GCE) instance with a public IP address. 
We streamline the instance configuration by providing an instance startup script.
It installs packages, disables AppArmor, and enables IP forwarding at the OS-level.

### WireGuard Server

A minimal overview of what we run on our VPN server.

- name : `wg-server`
- ens4 : `10.188.0.2` (or similar)
- wg0 : `172.16.42.1`
- Port: `51820` UDP
- [Caddy](https://caddyserver.com) (webserver) - to verify iPhone connectivity
- [Unbound](https://nlnetlabs.nl/projects/unbound/about/) (DNS resolver) - used by the iPhone 

### iPhone

- IP assigned from WireGuard server: `172.16.42.42`
- iOS Version tested `18.7.2`
- [WireGuard app](https://apps.apple.com/ca/app/wireguard/id1441195209) `1.0.16`

### Create GCE resources

We will create a startup script for our GCE instance, the GCE instance, and a firewall rule
to allow incoming UDP traffic on port `51820` to the instance.

#### GCE startup script

Create the startup script `wg-server-startup-script.sh` where you will run `gcloud compute instances create`.

```shell
#!/usr/bin/env bash
# WireGuard Server Startup Script
apt update
apt install -y \
  wireguard-go wireguard-tools qrencode unbound caddy apparmor-utils \
  iputils-ping net-tools iptables \
  bash-completion vim tmux
aa-disable /etc/apparmor.d/usr.sbin.unbound
systemctl stop apparmor
systemctl disable apparmor
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | tee -a /etc/sysctl.conf
sed -i 's/ip_forward=0/ip_forward=1/' /etc/sysctl.d/60-gce-network-security.conf
wireguard-go wg0
systemctl enable wg-quick@wg0.service
systemctl daemon-reload
```

#### GCE Instance Config

- Machine : `e2-small`
- Region: `northamerica-northeast2`
- OS: Ubuntu 25.10 minimal
- Disk: `10` Gbyte
- Cloud API access scopes: `cloud-platform`
```shell
gcloud compute instances create \
--machine-type=e2-small \
--image-family=ubuntu-minimal-2510-amd64 \
--image-project=ubuntu-os-cloud \
--scopes=cloud-platform \
--can-ip-forward \
--tags=vpn-server \
--metadata-from-file=startup-script=./wg-server-startup-script.sh \
--async \
wg-server
```

#### GCP VPC Firewall rule

The firewall rule allows UDP traffic to our port `51820` to our `wg-server`
GCE instance tagged with `vpn-server`.

```shell
gcloud compute --project=$CLOUDSDK_CORE_PROJECT \
firewall-rules create allow-wg \
--direction=INGRESS --priority=1000 --network=default --action=ALLOW \
--rules=udp:51820 --source-ranges=0.0.0.0/0 --target-tags=vpn-server
```
{{< rawhtml >}}
<!--
- Cloud Router
  ```shell
  gcloud compute routers create my-cloud-router \
  --network=default \
  --asn=65000
  ```
- Cloud NAT
  ```shell
  gcloud compute routers nats create my-nat \
  --router=my-cloud-router \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges
  ```
-->
{{< /rawhtml >}}

## Ubuntu Prerequisites

We need to install wireguard-go and tools.
Then, we set up our VPN server and generate the QR code for the mobile device.

### Prepare the Instance

Connect to the GCE instance.

```shell
# replace $MYPROJECT with your actual GCP project or set the variable
gcloud compute ssh \
  --tunnel-through-iap --project $MYPROJECT \
  wg-server
```

After the reboot, we will create the VPN server and client configuration files.

## Configure Interfaces and Wireguard

Next, we configure our wireguard interface.
This time, we create the configuration files and use `wg-quick` to spin up the interface.

### Configure Network Interfaces

Create a new network interface named `wg0` with type `wireguard` and generate the cryptographic key parts.

```shell
# Add wg0 network interface of type wireguard
wireguard-go wg0

# generate cryptographic key parts for server and iphone
cd /etc/wireguard
umask 077
wg genkey | tee server-privatekey | wg pubkey > server-publickey
wg genkey | tee iphone-privatekey | wg pubkey > iphone-publickey
```

Write the `wg0.conf` and `iphone.conf` files.
The server configuration includes masquerading rules to forward traffic
that is incoming from the tunnel to the internet.
Later, we will test that our phone can actually still browse :blush:

```shell
# set keys as environment variables
server_priv=$(<"server-privatekey")
server_pub=$(<"server-publickey")
server_wg_ip="172.16.42.1"
server_ext_ip=$(gcloud compute instances describe wg-server --zone=northamerica-northeast2-a --format='get(networkInterfaces[0].accessConfigs[0].natIP)')
iphone_priv=$(<"iphone-privatekey")
iphone_pub=$(<"iphone-publickey")
iphone_wg_ip="172.16.42.42/32"

cat <<EOF > wg0.conf
### WireGuard VPN Server
[Interface]
# IP range for client devices
Address = $server_wg_ip/24
ListenPort = 51820
# server private key
PrivateKey = $server_priv
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens4 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens4 -j MASQUERADE

[Peer]
# for iPhone
PublicKey = $iphone_pub
# IP to assign the iPhone
AllowedIPs = $iphone_wg_ip
EOF

cat <<EOF > iphone.conf 
### iphone client
[Interface]
# iphone private key
PrivateKey = $iphone_priv
Address = $iphone_wg_ip
DNS = 172.16.42.1
 
[Peer]
# WG server public key
PublicKey = $server_pub
Endpoint = $server_ext_ip:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 15
EOF
```

View the current wireguard configuration.

  > :bulb: In my case, a reboot was in order. Without it, `wg` would show me the error:
    `Unable to list interfaces: Permission denied`

```shell
# View wireguard config
wg

# example output
interface: wg0
  public key: pAhfFR80M5ZHpgkFrmSPt9hy519F39QbLUtIodlG0X8=
  private key: (hidden)
  listening port: 51820

peer: okNQnBt1xpJoDztwPQ8op9fRzbOxAbEf5dNV1lS9bR4=
  allowed ips: 172.16.42.42/32
```

### DNS Resolver

Configure the VPN server as a DNS resolver with [unbound](https://nlnetlabs.nl/projects/unbound/about/).

```shell
# unbound config
cat <<EOF > /etc/unbound/unbound.conf.d/wg.conf
server:
  access-control: 172.16.42.0/24 allow
  interface: 127.0.0.1
  interface: 172.16.42.1
  logfile: "/var/log/unbound/unbound.log"
  verbosity: 1
  log-queries: yes
EOF

# create the log dir and file
mkdir -p /var/log/unbound/
chown unbound: /var/log/unbound/
touch /var/log/unbound/unbound.log
chown unbound: /var/log/unbound/unbound.log
chmod 640 /var/log/unbound/unbound.log

# restart the service
systemctl restart unbound

# verify
netstat -tulpn
```

### QR codes

Generate the QR code for your VPN client (iPhone).
The QR code will show in your terminal.
You can find a video in the next section.

```shell
# Scan the code to connect your phone to the VPN server
qrencode -t ansiutf8 < iphone.conf
```

Before we change to the iPhone, let's create another QR code for our simple web page
hosted with [Caddy](https://caddyserver.com) on our VPN server.
This way, we can easily confirm access to resources on our server.

```shell
# Once we have iPhone to VPN server connected, scan this code to ensure
# we can see the Caddy sample web page from our VPN server
qrencode -t ansiutf8 "http://172.16.42.1:80/"
```

It is time! :hourglass_flowing_sand:

Let's connect to our VPN server and surf :surfer: the interwebs :spider_web: through our tunnel :star_struck:.

## Connect your iPhone

Install the [WireGuard app](https://apps.apple.com/ca/app/wireguard/id1441195209)
on your phone.

- Open the app
- Tap the :heavy_plus_sign: "Plus" sign
- Tap "Create from QR code"
- Scan the QR code
- Name the tunnel and tap "Save"
- Tap the profile slider to enable the tunnel

Scan the other (smaller) QR code with your Camera app to open your Caddy
sample web page, which is hosted on your VPN server.

Finally, try browsing some public site.

This is what it looks like on the iPhone.

iPhone Video:

{{< video src="wireguard-go-iphone" >}}

Sweet!

## Summary

It is pretty easy to set up your personal VPN server for your mobile
devices. Together with the unbound DNS resolver, we have a pretty
solid combination that doesn't require much configuration.
This setup could be enhanced to be closer to the Pi-hole project, for example,
to have our VPN server also act as a [DNS sinkhole](https://en.wikipedia.org/wiki/DNS_sinkhole).

Take care on the interwebs!
