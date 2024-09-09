+++
title = "Deploying & Securing a VPS"
description = "Virtual private server for self-hosting services and testing"
date = 2022-03-24
+++

&nbsp;

Having a VPS may unlock numerous possibilities regarding the usage of online services for ordinary _internautes_.
I believe anyone who is somewhat tech-savvy should get one, provided that she can afford it.
This is a short guide that goes over the basic steps to quickly set up a VPS - either on OpenBSD or on Linux - and make it secure so that we can reliably run various services on it.

---

First of all, we need to choose a VPS hosting provider to host our server and make an account.

After creating an account, we can provide the necessary payment info and deploy a new server with the preferred configuration.

The graphical Web interfaces are fairly straightforward and easy to navigate, but I will mention a few key points regarding the properties of the server.

+ For your first VPS, choose the cheapest plan available.
This should give you a single core of shared vCPU, 512 or 1024 MB of RAM, some storage and bandwidth.
You don't need a lot of resources for hosting simple services, and you can always upgrade later or use a snapshot to redeploy.

+ The server location should not matter a whole lot, but it might be a good idea to choose a location that is close to you, or to other potential users who might connect to the server.

+ As for the operating system, I would highly recommend OpenBSD if you just want a simple and secure system to work with and host your website as well as other services without hassle.
It is very pleasant and a great choice to become familiar with how a UNIX system works.

+ If you want to use Linux or depend on software that runs on Linux, I would recommend that you go with Fedora if you need a fairly stable system with the latest software, or Rocky Linux which is downstream of Fedora and is considered to be even more stable.
Debian and Ubuntu should also be mentioned as they are the most popular distributions.

+ Choose the latest OS version, unless you have a reason not to.

+ You may disable auto backups if you don't wish to pay extra for them.
At your own risk, of course.

+ You may still enable IPv6 as an additional option.

+ Choose a logical hostname and a label.

<div class="note"><small><b>Note:</b>
Make sure that the server size you choose is not for IPv6-only.
IPv4 is the standard for now, and you will need it.
</small></div>

After creating the server with these settings, information about the server including the public IP address and the root password should be available in the user panel of our VPS provider.
It might take a few minutes to finish running the initialization scripts, and we should be able to log in to the server after that.

In order to log in remotely and execute commands, we will be using `ssh`.
[OpenSSH](https://www.openssh.com/) should be pre-installed on most Linux, BSD and MacOS systems, and on more recent versions of Windows.
We can log in to the new server as the superuser using the public IP (Entering the root password when prompted).

```sh
user@void: ~$ ssh root@<public-IP>
```

The server is now accessible. We can now do the following things to improve security.

+ Create a regular user on the system and disallow remote root logins (use `su` instead)

+ Create an SSH keypair and log in as the user with an authorized key.

+ Change the default SSH port in order to mitigate automated bruteforce attacks.

<div class="note"><small><b>Note:</b>
Even though these attacks may not succeed, they will at least fill your firewall and sshd logs with hundreds of lines of bruteforce attempts from IPs in China.
It is best to change the default SSH port just to have cleaner logs.
</small></div>

We start by creating a regular user with a home directory.

```sh
root@server: ~# useradd -m username
```

<div class="note"><small><b>Note:</b>
The username here is "username", but this is for demonstration purposes.
You should ideally set a username that is harder to guess.
</small></div>

We set a password for the new user.

```sh
root@server: ~# passwd username
```

We also add the user to the "wheel" group for `su`.

```sh
root@server: ~# usermod -G wheel username
```

We now go back to our client PC and create a keypair, preferably with the ed25519 algorithm. We may also add a comment to help identify it.

```sh
user@void: ~$ ssh-keygen -t ed25519 -C "My key"
```

<div class="note"><small><b>Note:</b>
It is possible to change the name/location of the key when prompted, which will be helpful when creating multiple keys for different purposes and organizing them in a coherent manner.
I will change it to <code>.ssh/id_ed25519_0</code>.
</small></div>
&nbsp;
<div class="note"><small><b>Note:</b>
It is beneficial to also set a password for the key when prompted.
This provides an additional layer of security.<br><br>
Losing the keyfile or the password to it will lock you out of the server, so it is important to make sure that they are safe.
</small></div>

We can add our public key to `.ssh/authorized_keys` in the remote user's home directory manually, but `ssh-copy-id` provides an easier way to copy keys to a remote host.

```sh
user@void: ~$ ssh-copy-id -i ~/.ssh/id_ed25519_0.pub username@<public-IP>
```

We need to make a few changes in the `/etc/sshd_config` file on the server.
Log in now as a regular user, and use `su` to gain root access.

```sh
user@void: ~$ ssh username@<public-IP> -i ~/.ssh/id_ed25519_0
```

```sh
username@server: ~$ su
```

```sh
root@server: ~# vi /etc/ssh/sshd_config
```

We will disallow remote root logins and disable all password logins/interactive keyboard logins. This leaves public key authentication as our only authentication method when logging in.
We may also change the default SSH port now (to 44400 in this example).
We find, uncomment or add the following lines, adjusting the settings as shown below.

```sh
Port 44400
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
```

<div class="note"><small><b>Note:</b>
Depending on how your system was set up, there might be files inside the <code>/etc/ssh/sshd_config.d/</code> directory overriding the settings we just defined in the <code>/etc/ssh/sshd_config</code> file.
Make sure to check the contents of this directory and delete files/lines where necessary.
</small></div>

`sshd` needs to be restarted so that the changes take effect.

<div class="note"><small><b>Note:</b>
<b style="color: red;"><u>Do not</u></b> restart <code>sshd</code> yet if you have a firewall running.
You might get locked out if the new port is not open.
Temporarily disable the firewall, or check the firewall section of this guide first.
</small></div>

```sh
root@server: ~# rcctl restart sshd
```

<div class="note"><small><b>Note:</b>
<code>rcctl</code> is for OpenBSD.
Use <code>systemctl restart sshd</code> on Fedora (and on all Linux distributions with systemd).
</small></div>

It is also possible to use a config file on our client for convenience.

```sh
user@void: ~$ vim ~/.ssh/config
```

We write the following entry and save the file.

```bash
Host server
    HostName <public-IP>
    User username
    Port 44400
    IdentityFile ~/.ssh/id_ed25519_0
```

We can now log in simply like this:

```sh
user@void: ~$ ssh server
```

Finally, we can set some firewall rules to further improve security. This part is platform-specific.

<div class="note"><small><b>Note:</b>
These are just some very basic rules for firewalls.
This is not a detailed firewall guide in any way.
</small></div>

### Firewall for OpenBSD

Firewalling on OpenBSD is handled by [pf](https://www.openbsd.org/faq/pf/). All connections are passed by default. Rules can be defined in `/etc/pf.conf`.

```sh
root@server: ~# vi /etc/pf.conf
```

Here is a crude ruleset with some comments.

```ini
# Define the ssh port as a variable
port_ssh = "{44400}"

# Do not filter on the loopback interface
set skip on { lo }

# Drop packets that are blocked
# This gives less information to potential attackers
# Also saves bandwidth
set block-policy drop

# Block everything by default
block drop all

# Allow limited ICMP traffic (bleh)
pass in log on egress proto icmp max-pkt-rate 5/1

# Allow incoming ssh connections
pass in log on egress proto tcp from any to any port $port_ssh

# Allow all outgoing connections
pass out on egress from any to any
```

<div class="note"><small><b>Note:</b>
Pay attention to the lines concerning ssh. You might lock yourself out of the server with a typo.
</small></div>

`pf` should be enabled by default. We check the configuration and then load the new ruleset.

```sh
root@server: ~# pfctl -nf /etc/pf.conf
root@server: ~# pfctl -f /etc/pf.conf
```

### Firewall for Fedora/Linux

Firewalld is the default firewall program on Fedora and other Linux distributions related to the RHEL ecosystem, and it uses `nftables` as a backend.
On other distributions with systemd, `firewalld` can be install through the package manager.
Using `ufw` or using `iptables`/`nftables` directly are other options for firewalling on Linux, but this guide only covers some basic commands with `firewalld`.

Here is how we can set up basic firewall functionality using `firewalld`'s command-line interface: `firewall-cmd`.

First of all, we check if `firewalld` is running.

```sh
root@server: ~# systemctl status firewalld
```

If not, we enable and start it.

```sh
root@server: ~# systemctl enable --now firewalld
```

We can check the current configuration with:

```sh
root@server: ~# firewall-cmd --list-all
```

This lists the active _zones_ and their configurations. In `firewalld`, a network interface cannot be assigned to more than one zone. The external interface of your server should be assigned to the "public" zone, or some other custom zone. We can verify this by reading the output of the command above; under the zone name, it should contain a line that looks like:

```txt
    interface: enp1s0
```

In this case, `enp1s0` is the external interface. We can make sure that the zone of this interface (I am assuming that it is "public", but it could be something else on your system) is the default one for our commands.

```sh
root@server: ~# firewall-cmd --get-default-zone
```

If not:

```sh
root@server: ~# firewall-cmd --set-default-zone public
```

Now that we have the active zone linked to our external interface as our default zone, we can proceed.
`firewall-cmd` will show us the services and the ports that are open.

```sh
root@server: ~# firewall-cmd --list-all
```

Read the output, and look for lines starting with "services" and "ports".

```txt
    services: cockpit dhcpv6-client ssh
    ports:
```

First, we can remove the services that we don't need.
Cockpit is a Web interface for servers.
Feel free to keep it enabled, but I will remove it as an example, since it is not necessary.
Here is how you remove services with `firewall-cmd`.

```sh
root@server: ~# firewall-cmd --remove-service=cockpit
```

Next, we add our alternative ssh port (44400) to the list of allowed ports.
On Fedora, [SELinux](https://www.redhat.com/en/topics/linux/what-is-selinux) might prevent ssh connections on different ports, so we need to let it know that we are explicitly allowing ssh on port 44400.

The default port for ssh is port 22.

```sh
root@server: ~# semanage port -l | grep ssh
```

Output:

```txt
ssh_port_t                      tcp      22
```

We allow the new port:

```sh
root@server: ~# semanage port -a -t ssh_port_t -p 44000
```

And confirm that the new port was added:

```sh
root@server: ~# semanage port -l | grep ssh
```

The output should contain the new port.

```txt
ssh_port_t                      tcp      44000, 22
```

We can now open the port with `firewall-cmd`.

```sh
root@server: ~# firewall-cmd --add-port=44000/tcp
```

We may check if everything was configured the way we intended.

```sh
root@server: ~# firewall-cmd --list-all
```

When satisfied, we can make the configuration persistent across reboots.

```sh
root@server: ~# firewall-cmd --runtime-to-permanent
```

---

Here are a few things you can do on your VPS (Updated list):

+ Host a website ([OpenBSD httpd](@/handbook/httpd.md)).

+ [Set up a VPN and a DNS resolver](@/handbook/wireguard.md).

+ [Run a VoIP server](@/handbook/umurmur.md) for voice chat.
