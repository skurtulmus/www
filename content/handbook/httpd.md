+++
title = "Hosting a Website (with OpenBSD httpd)"
description = "Configuring a Web server, and a linked domain with HTTPS"
date = 2022-03-26
+++

&nbsp;

Having a personal website is one of the best ways to fine-tune your online presence as it allows you to have complete control over the content that you share, and the ways that you might want to structure it.
The base install of OpenBSD comes with its own Web server called `httpd`, and it is fairly simple to set up.
In this guide we will serve a webpage with `httpd`, register and link a domain name, and configure HTTPS (secure/encrypted HTTP) with `acme-client`

---

After [deploying a VPS](@/handbook/vps.md) (OpenBSD), we can log in with `ssh` and start setting up the server. We will need superuser access.

```sh
user@server ~$ su
Password:
root@server ~#
```

We begin by making a directory for our webpage inside the default chroot directory for the `httpd` Web server, which is `/var/www`. We also make a rudimentary home page.

```sh
root@server ~# mkdir -p /var/www/htdocs/example.net
root@server ~# vi /var/www/htdocs/example.net/index.html
```

Here is a skeletal HTML page that we can use for the purposes of this guide.

```html
<!DOCTYPE html>
<html>
    <head>
        <title>Home Page</title>
    </head>
    <body>
        <p>Hello World!</p>
    </body>
</html>
```

<div class="note"><small><b>Note:</b>
This file is just something to serve on the Web for now.
Creating a real functional webpage will involve expanding this with more files and some directory structure.
</small></div>

Now we open the configuration file of `httpd`.

```sh
root@server ~# vi /etc/httpd.conf
```

We edit the file to include the following lines.

```perl
# HTTP

server "example.net" {
    alias "www.example.net"
    listen on * port 80
    root "/htdocs/example.net"
}
```

<div class="note"><small><b>Note:</b>
Port 80 is the default port for HTTP. Port 443 is for HTTPS.
</small></div>

Assuming we have `pf` enabled and running, we need to open ports 80 and 443. We add the following lines to the existing configuration in `/etc/pf.conf`:

```ini
# Define HTTP/HTTPS ports as a variable
port_http = "{80,443}"

# Allow incoming HTTP/HTTPS connections
pass in log on egress proto tcp from any to any port $port_http
```

And load the new rules.

```sh
root@server ~# pfctl -f /etc/pf.conf
```

After checking the configuration, we enable and start `httpd`.

```sh
root@server ~# httpd -n
root@server ~# rcctl enable httpd
root@server ~# rcctl start httpd
```

The webpage can now be accessed by typing in the public IP address of our server into the URL bar of the Web browser.
But we want to be able to access it by simply typing "example.net".

### Linking a Domain Name

For the next step, we need to register a domain name.
I can recommend [Gandi](https://www.gandi.net) as a domain registrar.
[Namecheap](https://www.namecheap.com/) and [GoDaddy](https://www.godaddy.com) are other popular services.
Registering a domain is fairly straightforward once we provide the necessary information.

Having registered our domain, we navigate to the user panel/dashboard on our registrar's Web interface, and find the DNS records section.
There might be some preset records here.
We will delete all of them and add entries of our own.

Each record will have a record type, a subdomain name, a linked IP address, and a TTL (time to live) value for cached DNS information.
We will link our server's public IPv4 and IPv6 addresses to 3 records: The root (@) domain, "www" domain, and the wildcard (&ast;) for all other subdomains.
Type A is for IPv4, and type AAAA is for IPv6 addresses.
1800-3600 seconds should be reasonable values for TTL, but you can set it as low as 300 if you plan on changing IP addresses frequently.

<div class="note"><small><b>Note:</b>
Feel free to skip IPv6 records if you don't need them.
</small></div>

Here are the DNS record entries that we will create:

<table>
    <tr>
        <th>Type</th><th>Subdomain</th><th>TTL</th><th>IP Address</th>
    </tr><tr>
        <td>A</td><td>@</td><td>1800</td><td>public-IPv4-address</td>
    </tr><tr>
        <td>A</td><td>www</td><td>1800</td><td>public-IPv4-address</td>
    </tr><tr>
        <td>A</td><td>&ast;</td><td>1800</td><td>public-IPv4-address</td>
    </tr><tr>
        <td>AAAA</td><td>@</td><td>1800</td><td>public-IPv6-address</td>
    </tr><tr>
        <td>AAAA</td><td>www</td><td>1800</td><td>public-IPv6-address</td>
    </tr><tr>
        <td>AAAA</td><td>&ast;</td><td>1800</td><td>public-IPv6-address</td>
    </tr>
</table>

We should now be able to access our website by typing in the domain name (example.net).

### Configuring HTTPS

Finally, we can obtain certificates and configure the HTTPS connection.
The OpenBSD base install includes a tool to handle this task called `acme-client`, as well as an example configuration file.
We can copy this example file to the default configuration file location, and edit the "domain" section to include our domain.

```sh
root@server ~# cp /etc/examples/acme-client.conf /etc/acme-client.conf
root@server ~# vi /etc/acme-client.conf
```

The file should at least contain the following:

```perl
authority letsencrypt {
        api url "https://acme-v02.api.letsencrypt.org/directory"
        account key "/etc/acme/letsencrypt-privkey.pem"
}

domain example.net {
        alternative names { www.example.net }
        domain key "/etc/ssl/private/example.net.key"
        domain full chain certificate "/etc/ssl/example.net.fullchain.pem"
        sign with letsencrypt
}
```

<div class="note"><small><b>Note:</b>
The example file includes more authority declarations, but we do not use them. They do not have to be removed.
</small></div>

We also need to adjust `httpd.conf` to accommodate HTTPS connections.
Here's what the new configuration will look like.

```perl
# HTTP/HTTPS

server "example.net" {
    alias "www.example.net"
    listen on * port 80
    listen on * tls port 443
    tls {
        certificate "/etc/ssl/example.net.fullchain.pem"
        key "/etc/ssl/private/example.net.key"
    }
    root "/htdocs/example.net"
    location "/.well-known/acme-challenge/*" {
        root "/acme"
        request strip 2
    }
}
```

We quickly reload `httpd` so that the configuration is applied.

```sh
root@server ~# rcctl reload httpd
```

Next, we make sure that the ACME challenge directory and the certificate directory that we used in the configuration files exist (with correct permissions).

```sh
root@server ~# mkdir -p /var/www/acme
root@server ~# chmod 755 /var/www/acme
root@server ~# mkdir -p /etc/ssl/private
root@server ~# chmod 700 /etc/ssl/private
```

We are now ready to generate certificates.

```sh
root@server ~# acme-client -v example.net
```

Reload `httpd`, and everything should be working. Our website can be accessed at "https://example.net"

### Automatic Certificate Renewal

The certificate is valid for 90 days. We can make a `crontab` entry to run `acme-client` regularly and make certificate renewal automatic.

```sh
root@server ~# crontab -e
```

We make the following entry to run `acme-client` and reload `httpd` at the start of every month.

```sh
0     0     1     *     *     acme-client -v example.net && rcctl reload httpd
```
