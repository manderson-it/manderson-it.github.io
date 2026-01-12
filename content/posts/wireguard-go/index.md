+++
title = 'WireGuard in Userspace'
date = 2026-01-12T11:02:09-05:00
draft = true
+++
## WireGuard Continued

In the [previous article](https://manderson-it.ca/posts/wireguard-fun/), we set up a VPN tunnel with WireGuard between two sites.

{{< rawhtml >}}
<!--
https://www.youtube.com/watch?v=NFRUN5FwhY0
-->
{{< /rawhtml >}}

Here, we apply a userspace implementation of WireGuard, set up our VPN server, and connect from an iPhone.
Why would one want to use a userspace implementation though?

Generally, we want to use WireGuard as a kernel module for best raw performance.
The kernel module will be available in a lot of cases.
You may find yourself working with a system that doesn't offer kernel module support.
This could apply to: older or locked-down OSes, some BSD variants, minimal containers or unikernels, or embedded systems.
Similarly, you may need WireGuard as a library if you want to embed it inside an application, you need programmatic control over tunnels, or build a custom vpn client.
In other words, if you are working in a restricted environment, or flexibility and portability are more important, running WireGuard as a userspace process can be a good choice.

Let us choose the well established Golang implementation, called wireguard-go.
Its [git repository is here](https://git.zx2c4.com/wireguard-go), and some
[explanation around userspace implementation is on the project site](https://www.wireguard.com/xplatform/).

What we will build is as straightforward as the diagram below.
We will create the VPN server, generate a QR code for our client configuration, and scan the QR code
on an iPhone to establish the VPN connection.

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
      BA([fa:fa-phone iPhone])
      QR\-\->|fa:fa-camera|BA
      QR([fa:fa-qrcode QR code])
    end
    subgraph "fab:fa-google GCP"
      CB([fab:fa-fort-awesome-alt WireGuard server])
    end
    BA -\-\->|fas:fa-cloud tunnel| CB
-->
{{< /rawhtml >}}

[![Architecture of the Hamburg and Berlin sites in GCP](/images/wireguard-go-arch.png)](/images/wireguard-go-arch.png)

## My Environment

You need two Google Compute Engine (GCE) instances with no firewall between them. It is for demonstration purposes only.

  - WireGuard Server
    - ens4 : `10.128.0.2`
    - wg0 : `192.168.2.1`
  - iPhone
    - iOS Version `18.7.2`
    - [WireGuard app](https://apps.apple.com/ca/app/wireguard/id1441195209) `1.0.16`
  - Config
    - Machine : `e2-small`
    - Region: `us-central1`
    - OS: Ubuntu 25.10 minimal
    - Disk: `10` Gbyte
    - VPC Firewall rule
      ```shell
      gcloud compute --project=$MYPROJECT \
      firewall-rules create allow-wg \
      --direction=INGRESS --priority=1000 --network=default --action=ALLOW \
      --rules=udp:51820 --source-ranges=0.0.0.0/0
      ```
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

Stop AppArmor, install packages, and enable IP forwarding.

```shell
apt update
apt install -y \
  wireguard-go wireguard-tools qrencode \
  iputils-ping net-tools iptables \
  bash-completion vim tmux
systemctl stop apparmor
echo "net.ipv4.ip_forward=1" | tee -a /etc/sysctl.conf
# ensure bash completion works
exit
sudo su -
```

## Configure Interfaces and Wireguard

Next, we configure our wireguard interface.
This time, we create the configuration file and use `wg-quick` to spin it up.

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

TODO: change `iptables` to `nft`

```shell
cat <<EOF > wg0.conf
### WireGuard VPN Server
[Interface]
# IP range  for client devices
Address = 172.16.42.1/24
ListenPort = 51820
# server private key
PrivateKey = secret
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens4 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens4 -j MASQUERADE

[Peer]
# for iPhone
PublicKey = secret
# IP to assign the iPhone
AllowedIPs = 172.16.42.42/32
EOF

cat <<EOF > iphone.conf 
### iphone client
[Interface]
# iphone private key
PrivateKey = secret
Address = 172.16.42.42/32
DNS = 8.8.8.8
 
[Peer]
# WG server public key
PublicKey = secret
Endpoint = 34.130.150.46:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 15
EOF
```

View the current wireguard configuration.

```shell
# View wireguard config
wg
# equivalent to
wg show

# example output
interface: wg0
  listening port: 51820
```

Define the VPN tunnel on `Hamburg`. Ensure the port matches the `wg` output from the peer.

```shell
# On Hamburg
#
# As arguments, use the public_key, wg0 IP, and ens4 IP from peer-b
wg set wg0 peer 9O/Wm3NJeinXGKk5s6sqtOS/rKWf7z45Nc2mFRecKUw= allowed-ips 192.168.2.2 endpoint 10.128.0.3:51820

# View configuration
wg

# example output
interface: wg0
  listening port: 51820

peer: 9O/Wm3NJeinXGKk5s6sqtOS/rKWf7z45Nc2mFRecKUw=
  endpoint: 10.128.0.3:51820
  allowed ips: 192.168.2.2/32
```

Generate the QR code for your iPhone.

```shell
qrencode -t ansiutf8 < iphone.conf
```

Ping the iPhone through the tunnel to confirm reachability.

```shell
# On server
ping 172.16.42.42
PING 172.16.42.42(172.16.42.42) 56(84) bytes of data.

```

Finally, inspect the tunnel status with wireguard.

```shell
# show tunnel status with wireguard
wg

# example output
interface: wg0
  public key: rXiDojveu4Q6VVIzw2Pm1LFBJLbeWSV6rc/MQqcpWlE=
  private key: (hidden)
  listening port: 51820

peer: 9O/Wm3NJeinXGKk5s6sqtOS/rKWf7z45Nc2mFRecKUw=
  endpoint: 10.128.0.3:51820
  allowed ips: 192.168.2.2/32
  latest handshake: 1 minute, 35 seconds ago
  transfer: 1.05 KiB received, 1.23 KiB sent
```

Here is the video of the steps above.

WG Server Video:

This is what it looks like on the iPhone.

iPhone Video:

## Reflection So Far

Well, that was pretty easy and straightforward 😄.

We did have to type quite a few commands though. Here is a summary of the WireGuard-related steps.
We used:

- three `ip` commands to bring up the network interface, and
- three `wg` commands to configure WireGuard

```shell
# summary of WireGuard-related commands
modprobe -v wireguard
ip link add dev wg0 type wireguard
ip address add dev wg0 $WGIP/24
cd /etc/wireguard
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
wg set wg0 private-key /etc/wireguard/privatekey listen-port 51820
ip link set up dev wg0
wg set wg0 peer 9O/Wm3NJeinXGKk5s6sqtOS/rKWf7z45Nc2mFRecKUw= allowed-ips 192.168.2.2 endpoint 10.128.0.3:51820
```

Can the commands be streamlined after our one-time setup?

## WireGuard Provides

It is quick and easy setting up the tunnel interface with `wg-quick`.
Now, that we have our configuration, we can save it to a file.
That way, we can quickly bring up the tunnel with a single command.

```shell
# save current wg configuration to a file
cd /etc/wireguard
touch wg0.conf
wg-quick save wg0
```

The `wg-quick` tool allows us to quickly bring our interface `wg0` up/down.
You can explore this with the following commands.

```shell
# bring the interface down
wg-quick down wg0
# example output
[#] ip link delete dev wg0

# verify
ip addr show wg0

# bring it up
wg-quick up wg0
# example output
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 192.168.2.1/24 dev wg0
[#] ip link set mtu 1380 up dev wg0

# verify
ip addr show wg0
wg show
```

Neat! :star_struck:

And, if you want to add it as a systemd service:

```shell
sudo systemctl enable wg-quick@wg0.service
sudo systemctl daemon-reload
```

Cool, cool, cool! :nerd_face:

## Summary

WireGuard delivers what it promises.
It is a fantastic way to create a secure VPN tunnel these days!

If you are interested how WireGuard's performance compares to IPsec and OpenVPN, check [this](https://www.wireguard.com/performance/) out.
