---
layout: post
title: Setup wireguard VPN on linux
tags: linux
---
# Wireguard
There are a lot of blogs/documents explaining what wireguard is and how is it compared to IPSEC or OpenVPN. Some of them are explained in the below links.
Wireguard is a vpn that runs inside linux kernel. 

- https://www.wireguard.com/
- https://protonvpn.com/blog/openvpn-vs-wireguard/
- https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-20-04

## Setup wireguard
The below has been tested on Ubuntu on an x86 machine

### Packages needed
Packages: wireguard wireguard-tools wireguard-dkms
```
root@wireguard:/# apt-get install software-properties-common

apt install wireguard

```
### Check Status
```
root@wireguard:/etc/wireguard# dkms status
wireguard, 1.0.20200429, 4.15.0-99-generic, x86_64: installed
```

## Generate public and private key

### Public key
```
umask 077
ug genkey|tee privatekey|wg pubkey>publickey
```

### Private key
```
wg genkey > private
wg pubkey < private
```

## Setup the wg vpn interface 
```
sudo ip link add wg0 type wireguard
sudo ip addr add 191.168.2.2/24 dev wg0
sudo wg set wg0 private-key ./private listen-port 51820
sudo ip link set wg0 up
```
### Set peer
```
sudo wg set peer Qk2h0l25tsyV79PLu1Lu8CzVpUjtPJdKmLMmgRm1U0s= allowed-ips 191.168.2.1/32 
```
### Validate
```
#ifconfig wg0

#wg show
interface: wg0
  public key: gUxf0SHnVdRR4voEezgydoBVWO4/vvWLZ/PhAr0HGEE=
  private key: (hidden)
  listening port: 51820

peer: Qk2h0l25tsyV79PLu1Lu8CzVpUjtPJdKmLMmgRm1U0s=
  endpoint: w.x.y.z:25784
  allowed ips: 0.0.0.0/0, ::/0
  latest handshake: 18 seconds ago
  transfer: 23.34 MiB received, 92.99 MiB sent
```

## Creating server config
The above can be created as .conf file in the location `/etc/wireguard/wg0.conf`
```
/etc/wireguard/wg0.conf
[Interface]
PrivateKey = base64_encoded_peer_private_key_goes_here
Address = 10.8.0.2/24
Address = fd0d:86fa:c3bc::2/64

[Peer]
PublicKey = U9uE2kb/nrrzsEU58GD3pKFU3TLYDMCbetIsnV8eeFE=
AllowedIPs = 10.8.0.0/24, fd0d:86fa:c3bc::/64
Endpoint = 203.0.113.1:51820
```
Note: The above is from [digital ocean](https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-20-04). The document is great to get started on how to setup wireguard.

## Accessing your home LAN
To actually access the server’s LAN, you’ll need to make a slight modification to the configuration. On the server’s config file, at the end of the the [Interface] section, add these two lines:

```
PostUp   = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```
