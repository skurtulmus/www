+++
title = "Linux/OpenBSD Cheatsheet"
description = "Useful Linux commands and their OpenBSD equivalents"
date = 2023-11-04
+++

&nbsp;

Linux and OpenBSD do share a great deal of common commands and core utilities, albeit with different implementations.
However, there are some obvious differences, due to the ways these systems work.
This page contains a cheatsheet for common commands, operations and files on OpenBSD and Linux (primarily Red Hat family) systems, which covers some of these differences.


---

&nbsp;

## General Commands

<table style="width: 100%; table-layout: fixed; font-size: 115%;">
    <tr>
        <th style="color: #88ccff;">Linux</th><th style="color: #ffaa66;">OpenBSD</th>
    </tr><tr>
        <td style="opacity: 75%;"><code>sudo</code></td><td style="opacity: 75%;"><code>doas</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>sudo su</code></td><td style="opacity: 75%;"><code>doas su</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>lsof</code></td><td style="opacity: 75%;"><code>fstat</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>lsblk</code></td><td style="opacity: 75%;"><code>sysctl hw.disknames</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>ip a</code></td><td style="opacity: 75%;"><code>ifconfig</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>lspci</code></td><td style="opacity: 75%;"><code>pcidump</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>adduser &lt;username&gt;</code></td><td style="opacity: 75%;"><code>useradd -m &lt;username&gt;</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>deluser &lt;username&gt;</code></td><td style="opacity: 75%;"><code>userdel &lt;username&gt;</code></td>
    </tr>
</table>

## Service Management

<table style="width: 100%; table-layout: fixed; font-size: 115%;">
    <tr>
        <th style="color: #88ccff;">Linux</th><th style="color: #ffaa66;">OpenBSD</th>
    </tr><tr>
        <td style="opacity: 75%;"><code>systemctl stop &lt;service&gt;</code></td><td style="opacity: 75%;"><code>rcctl stop &lt;service&gt;</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>systemctl start &lt;service&gt;</code></td><td style="opacity: 75%;"><code>rcctl start &lt;service&gt;</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>systemctl status &lt;service&gt;</code></td><td style="opacity: 75%;"><code>rcctl check &lt;service&gt;</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>systemctl reload &lt;service&gt;</code></td><td style="opacity: 75%;"><code>rcctl reload &lt;service&gt;</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>systemctl enable &lt;service&gt;</code></td><td style="opacity: 75%;"><code>rcctl enable &lt;service&gt;</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>systemctl disable &lt;service&gt;</code></td><td style="opacity: 75%;"><code>rcctl disable &lt;service&gt;</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>systemctl restart &lt;service&gt;</code></td><td style="opacity: 75%;"><code>rcctl restart &lt;service&gt;</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>systemctl list-units --type=service</code></td><td style="opacity: 75%;"><code>rcctl ls all</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>systemctl list-units --type=service --state=running</code></td><td style="opacity: 75%;"><code>rcctl ls started</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>systemctl list-units --type=service --state=enabled</code></td><td style="opacity: 75%;"><code>rcctl ls on</code></td>
    </tr>
</table>

## Firewall & Networking

<table style="width: 100%; table-layout: fixed; font-size: 115%;">
    <tr>
        <th style="color: #88ccff;">Linux</th><th style="color: #ffaa66;">OpenBSD</th>
    </tr><tr>
        <td style="opacity: 75%;"><code>systemctl reload firewalld</code></td><td style="opacity: 75%;"><code>pfctl -f /etc/pf.conf</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>systemctl enable firewalld</code></td><td style="opacity: 75%;"><code>pfctl -e</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>systemctl disable firewalld</code></td><td style="opacity: 75%;"><code>pfctl -d</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>firewall-cmd --list-all</code></td><td style="opacity: 75%;"><code>pfctl -sr</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>firewall-cmd --add-port=...</code></td><td style="opacity: 75%;"><code>vi /etc/pf.conf</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>firewall-cmd --runtime-to-permanent</code></td><td style="opacity: 75%;">-</td>
    </tr><tr>
        <td style="opacity: 75%;"><code>systemctl restart network</code></td><td style="opacity: 75%;"><code>sh /etc/netstart</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>less /var/log/firewalld</code></td><td style="opacity: 75%;"><code>tcpdump -n -e -ttt -r /var/log/pflog</code></td>
    </tr>
</table>

## Packages & Upgrades

<table style="width: 100%; table-layout: fixed; font-size: 115%;">
    <tr>
        <th style="color: #88ccff;">Linux</th><th style="color: #ffaa66;">OpenBSD</th>
    </tr><tr>
        <td style="opacity: 75%;"><code>yum/dnf install</code></td><td style="opacity: 75%;"><code>pkg_add</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>yum/dnf update</code></td><td style="opacity: 75%;"><code>pkg_add -u</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>yum/dnf remove</code></td><td style="opacity: 75%;"><code>pkg_delete</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>yum/dnf autoremove</code></td><td style="opacity: 75%;"><code>pkg_delete -a</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>yum/dnf search</code></td><td style="opacity: 75%;"><code>pkg_info -Q</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>-</code></td><td style="opacity: 75%;"><code>syspatch -c</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>-</code></td><td style="opacity: 75%;"><code>syspatch</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>system-upgrade/dist-upgrade</code></td><td style="opacity: 75%;"><code>sysupgrade</code></td>
    </tr>
</table>

## Important Files

<table style="width: 100%; table-layout: fixed; font-size: 115%;">
    <tr>
        <th style="color: #88ccff;">Linux</th><th style="color: #ffaa66;">OpenBSD</th>
    </tr><tr>
        <td style="opacity: 75%;"><code>.bashrc</code></td><td style="opacity: 75%;"><code>.kshrc</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>/etc/sudoers</code></td><td style="opacity: 75%;"><code>/etc/doas.conf</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>/etc/sysconfig/network-scripts/ifcfg-&lt;interface&gt;</code></td><td style="opacity: 75%;"><code>/etc/hostname.&lt;interface&gt;</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>/etc/firewalld/firewalld.conf</code></td><td style="opacity: 75%;"><code>/etc/pf.conf</code></td>
    </tr><tr>
        <td style="opacity: 75%;"><code>/var/log/secure</code></td><td style="opacity: 75%;"><code>/var/log/authlog</code></td>
    </tr>
    </tr><tr>
        <td style="opacity: 75%;"><code>/var/log/nginx</code></td><td style="opacity: 75%;"><code>/var/www/logs</code></td>
    </tr>
</table>

