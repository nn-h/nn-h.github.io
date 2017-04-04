---
layout: post
title: Linuxmint VPN Dropouts
---

First things first, I love Linuxmint. I think its a great distro and I prefer
the flavor of software that it offers against some of the other distros I've
tinkered with in the past. Sure I can add what I need and like the Arch savants would build from scratch, but I'm not there. Unfortunately with all things Unix I've encountered,
there are some things that don't work out of the box, regardless of how many different
ways you might set up the configuration, it still just doesn't work. I signed up to
give [CactusVPN](http://www.cactusvpn.com) a shot to keep traffic a little more under the radar. Since LM comes installed with NetworkManager for the xfce flavor I'm running to manage VPN networks, it's a pretty straight forward process to get connected to a VPN. However I ran into a hiccup after noticing that after short period of time the VPN would drop out.

After some investigating at `/var/log/syslog` I found the following:

```nm-openvpn[5112]: WARNING: Failed running command (--up/--down): external program exited with error status: 1
nm-openvpn[5112]: Exiting due to fatal error
NetworkManager: <warn> VPN plugin failed: 1```

After a little investigating I found a [bug report](https://bugzilla.gnome.org/show_bug.cgi?id=556134) that although old, provided a potential workaround. The solution, to provide a wrapper script to run before the script containing the bug,

  `/usr/lib/NetworkManager/nm-openvpn-service-openvpn-helper`


First backup the original,

  `sudo cp /usr/lib/NetworkManager/nm-openvpn-service-openvpn-helper
  /usr/lib/NetworkManager/nm-openvpn-service-openvpn-helper.orig`

Then replacing the original script with the following wrapper,

    #!/bin/bash

    [ -z "${ifconfig_local}" ] && export ifconfig_local=$4
    [ -z "${ifconfig_remote}" ] && export ifconfig_remote=$5

    /usr/libexec/nm-openvpn-service-openvpn-helper.orig $*


So what's happening here? `ifconfig_local` & `ifconfig_remote` when the --up-restart param
is being called are not explicitly defined resulting in nm-openvpn-service-openvpn-helper to exit with status code 1, and hence terminating the VPN connection. So instead we reassert these values before the script is called ensuring they are defined before openvpn-helper continues.

Since implementing the workaround the VPN connection has terminated on several occasions,
however checking with syslog shows that it instead attempts to restart correctly when the
service disappears.

```NetworkManager[983]: <info> VPN service 'openvpn' disappeared
NetworkManager[983]: <info> Starting VPN service 'openvpn'...
NetworkManager[983]: <info> VPN service 'openvpn' started (org.freedesktop.NetworkManager.openvpn), PID 8373
NetworkManager[983]: <info> VPN service 'openvpn' appeared; activating connections
<info> VPN plugin state changed: starting (3)
<info> VPN connection 'CactusVPN NL, Amsterdam4' (Connect) reply received.
```
