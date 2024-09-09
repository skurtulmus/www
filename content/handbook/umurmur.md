+++
title = "Installing and Running umurmur (for Mumble)"
description = "Running a VoIP server for voice chat"
date = 2023-08-20
+++

&nbsp;

Instead of relying on Discord's "dedicated" and "private" servers, we can run our own open-source private VoIP server once we have a [VPS running](@/handbook/vps.md).
This is, in fact, quite easy to achieve using a lightweight server component designed for Mumble, `umurmur`.

---

For this guide, we will be using an OpenBSD server, but feel free to follow similar steps on Linux.
First, we need to install `umurmur`, since it is not included in the base install.
A binary package is provided, which can be installed using the package manager.

```sh
root@server ~$ pkg_add umurmur
```

It is a good idea to edit the configuration file now to define a password (for users) and an admin password for the server.
16-character random passwords can be generated using the one-liner below.

```sh
root@server ~$ tr -dc '[:graph:][:punct:]' 2>&1 < /dev/urandom | dd bs=16 count=1 status=none 2>&1 && printf "\n"
```

We can now look for the `password` and the `admin_password` directives in the `/etc/umurur/umurmur.con` file, and insert our generated passwords.

```sh
root@server ~$ vi /etc/umurmur/umurmur.conf
```

```pl
password="...";
admin_password="...";
```

Feel free to make any other parameter changes that you see fit, or to add/remove/rename channels.
After saving the file, we can enable and start the `umurmurd` service to run the server.

```sh
root@server ~$ rcctl enable umurmurd
root@server ~$ rcctl start umurmurd
```

We have a running `umurmur` server!
We can now connect to it from Mumble clients and start voice chatting.
First, we need to install a Mumble client on a client machine.
This should be available on most Linux and BSD systems through the package management system.

### On OpenBSD

```sh
user@obsd ~$ doas pkg_add mumble
```

### On Linux (Red Hat/Fedora)

```sh
user@linux ~$ sudo dnf install mumble
```

### On Linux (Debian/Ubuntu)

```sh
user@linux ~$ sudo apt install mumble
```

### On Linux (Void)

```sh
user@linux ~$ sudo xbps-install mumble
```

Upon running the client for the first time, Mumble will go through the initial setup for sound settings.
Once the setup is complete, we open the server connections interface.

<div class="text_img"><img src="/assets/images/handbook/mumbleconnect.png"></div>

Then, we add a new server, and fill in the public IP address of our server.
The default port is `64738`, which is the port our server uses.
The username and the label can be set to anything.

<div class="text_img"><img src="/assets/images/handbook/mumbleparams.png"></div>

After adding the server, we click the __Connect__ button.

We are now connected to the server, and we can start chatting!
