+++
title = "Wireguard VPN for OpenBSD/Linux"
description = "Setting up an encrypted VPN tunnel with Wireguard"
date = 2022-06-19
+++

&nbsp;

Wireguard is a simple VPN (Virtual Private Network) protocol known for its speed and security.
It has been implemented in the Linux kernel as well as in OpenBSD.
This is a guide on how to set up a Wireguard connection between two devices (an OpenBSD server and a Linux client) and a DNS resolver, in order to securely tunnel the traffic while using insecure public networks, or in any other case where it might be needed.

---

The official website of Wireguard includes a [conceptual overview](https://www.wireguard.com/#conceptual-overview) section and other links where the protocol is explained in more detail.
What we need to do is generate keypairs for our server and our client (which are not taxonomically different in terms of how their configurations are handled), add the client as a peer on the server configuration, and add the server as a peer on the client configuration.
Multiple peers may be added on the server to allow more client connections.

We begin by generating a base64 encoded 32-byte string that Wireguard can use as our private key.

```sh
user@openbsd ~$ openssl rand -base64 32 > wg0.key
```

The private key is now stored in the `wg0.key` file. We change the file permissions for good measure.

```sh
user@openbsd ~$ chmod 600 wg0.key
```

Next, we create a network interface for Wireguard associating it with the private key we generated.
We can also change the port for some obscurity, instead of using the default which is 51820. I chose 50101 in this case.

```sh
user@openbsd ~$ doas ifconfig wg0 create wgport 50101 wgkey `cat wg0.key`
```

We can now verify that the interface was created with `ifconfig`.

```sh
user@openbsd ~$ doas ifconfig
```

The `wg0` interface should appear in the output.

```txt
wg0: ...
...
    wgpubkey: <server-public-key>
...
```

The output should contain the public key that is extracted from the private key, on the line starting with `wgpubkey`. We note down this key.

We also need to assign an IP address to the server for the VPN tunnel. I'm using the 10.0.0.0/24 subnet since it is available and reserved for private networks. You can use another subnet if this one not available for your network.

```sh
user@openbsd ~$ doas ifconfig wg0 10.0.0.1 netmask 255.255.255.0
```

Now we move on to the Linux client to set up the interface.

Here, we install the `wireguard-tools` package since it makes things easier. I am using Void Linux; you can use the package manager of your distribution.

```sh
user@void ~$ sudo xbps-install -Su wireguard-tools
```

<div class="note"><small><b>Note:</b> It is also possible to use <code>wireguard-tools</code> on OpenBSD, but unnecessary in my opinion since the OpenBSD implementation is very simple.</small></div>

We create a new Wireguard interface. This time we do not need to specify a wg port, as we do not need to handle incoming connections on the client for this setup.

```sh
user@void ~$ sudo ip link add dev wg0 type wireguard
```

We also assign a different IP address for this endpoint, again on the 10.0.0.0/24 subnet.

```sh
user@void ~$ sudo ip address add dev wg0 10.0.0.2/24
```

This time, we generate a key using the `wg` command from `wireguard-tools` and add it to the wg0 interface on our client. It is possible to use `openssl` for this as well.

```sh
user@void ~$ wg genkey > wg0.key
user@void ~$ chmod 600 wg0.key
user@void ~$ sudo wg set wg0 private-key wg0.key
```

Next up, we need to connect the client and the server.
For this step, we need the public keys in order to add peers.
We can check the public key that was extracted from the private key for our client.

```sh
user@void ~$ sudo wg
```

The public key appears in the output. We note this key down.

```txt
interface: wg0
    public key: <client-public-key>
    private key: (hidden)
```

First, we add our server as a peer on the client, using the server public key we obtained earlier.

```sh
user@void ~$ sudo wg set wg0 peer <server-public-key> allowed-ips 0.0.0.0/0 endpoint <server-public-IP>:50101
```

This specifies the peer public key, the allowed IP addresses and the server endpoint including the IP address and the port.

<div class="note"><small><b>Note:</b> <code>allowed-ips</code> is set to 0.0.0.0/0, which means all IP addresses. This tells Wireguard to send packages destined for any IP address over the tunnel. Therefore, all traffic while browsing the Internet goes through Wireguard.</small></div>

Now we write all of this to a configuration file on the Linux client, in order to be able to quickly set up the interface and initiate the connection in the future.

```sh
user@void ~$ sudo mkdir -p /etc/wireguard
user@void ~$ sudo vim /etc/wireguard/wg0.conf
```

Here is the configuration including all the information.

```ini
[Interface]
PrivateKey = <client-private-key>
Address = 10.0.0.2/24
DNS = 10.0.0.1

[Peer]
PublicKey = <server-public-key>
AllowedIPs = 0.0.0.0/0
Endpoint = <server-public-IP>
PersistentKeepalive = 25
```

<div class="note"><small><b>Note:</b> Notice the <code>DNS</code> line in the configuration file. We will be setting up a DNS resolver on our server with <code>unbound</code> later. This is recommended, since your queries will go over your ISP's default DNS servers otherwise, which defeats the purpose of using a VPN for privacy in most cases. If this is not what you want, you should omit this line so that you can keep using the default DNS settings.</small></div>

Next, we add the client as a peer on our OpenBSD server, and make the configuration persistent across reboots by creating a `hostname` file for the interface.

```sh
user@openbsd ~$ doas ifconfig wg0 wgpeer <client-public-key> wgaip 10.0.0.2/24
user@openbsd ~$ doas vi /etc/hostname.wg0
```

Here is what we should write in `hostname.wg0`.
```vim
inet 10.0.0.1 255.255.255.0
wgkey <server-private-key>
wgport 50101
wgpeer <client-public-key> wgaip 10.0.0.2/24
up
```

We can use `unbound` with a simple setup for now. We open the `unbound` configuration file.

```sh
user@openbsd ~$ doas vi /var/unbound/etc/unbound.conf
```

Here is a basic configuration.

```yaml
server:

  interface: 0.0.0.0
  interface: ::1

  access-control: 10.0.0.1/24 allow
  access-control: 127.0.0.0/8 allow
  access-control: ::1 allow

  access-control: 0.0.0.0/0 refuse
  access-control: ::0/0 refuse

  hide-identity: yes
  hide-version: yes

  auto-trust-anchor-file: "/var/unbound/db/root.key"
  qname-minimisation: yes

  aggressive-nsec: yes
```

Now we enable and start `unbound`

```sh
user@openbsd ~$ doas rcctl enable unbound
user@openbsd ~$ doas rcctl start unbound
```

We also need to enable IP forwarding (and make it persistent across reboots) on our OpenBSD server so that the VPN can function for browsing the Web.

```sh
user@openbsd ~$ doas sysctl net.inet.ip.forwarding=1
user@openbsd ~$ doas sysctl net.inet6.ip6.forwarding=1
user@openbsd ~$ doas echo "net.inet.ip.forwarding=1" >> /etc/sysctl.conf
user@openbsd ~$ doas echo "net.inet6.ip6.forwarding=1" >> /etc/sysctl.conf
```

Finally, we need to make the necessary changes in our `pf` firewall configuration in order to allow traffic over Wireguard.

```sh
user@openbsd ~$ doas vi /etc/pf.conf
```

We add the following lines to `pf.conf`.

```ini
port_wg = "{50101}"
pass in log on wg #allow traffic on wg interfaces
pass in log on egress inet proto { udp } from any to any port $port_wg #allow incoming traffic on the Wireguard port
match out on egress inet from (wg:network) to any nat-to (egress:0) #allow NAT (network address translation) on outgoing traffic
```

We load the firewall configuration.

```sh
user@openbsd ~$ doas pfctl -f /etc/pf.conf
```

That's is. We should now be able to initiate the VPN connection on our Linux client, and tunnel the traffic over Wireguard.

```sh
user@void ~$ sudo wg-quick up wg0
```
