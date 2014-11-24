---
title: Configuring a network bridge on Debian 8.0 for static addresses
layout: post
---

Setting up bridge-utils to work correctly is something I only have to do
once every so often. It is one of those things where you end up chasing
the same rabbit hole of documentation over and over again in order to get
it to work whilst banging your against the wall. The reason why bridged
networking can end up being so frustrating isn’t because it is difficult
(it’s actually pretty straight forward) but because if you mess up, it
could mean a nice trip down to where you parked the hardware. I mostly use
network bridging on physical machines in order [to connect qemu machines
to the internet](http://wiki.qemu.org/Documentation/Networking) using
[TUN/TAP](http://en.wikipedia.org/wiki/TUN/TAP). 

Messing up a bridge-utils configuration has resulted in me borking
networking on remote and headless machines more than once now. To prevent
myself (and maybe you) some future headaches and lock-outs, this is how to
configure a bridge by the name of br0 with static IPs for interfaces eth0
and tap0 on a fresh install of Debian 8.0 jessie/sid. 

Firstly make sure bridge-utils is on the system
{% highlight bash %}
# apt-get install bridge-utils
{% endhighlight %}

Then put something like this based on your network preferences in
/etc/network/interfaces.
I recommend commenting out the original configuration lines instead of
deleting them.
{% highlight bash %}
iface lo inet loopback

allow-hotplug eth0
iface eth0 inet static
        address 192.168.1.155	#eth0 ip
        netmask 255.255.255.0
        gateway 192.168.1.1

allow-hotplug tap0
iface tap0 inet static
        address 192.168.1.4	#tap0 ip
        netmask 255.255.255.0
        gateway 192.168.1.1

auto br0
iface br0 inet static
        address 192.168.1.2	#bridge ip
        netmask 255.255.255.0
        broadcast 192.168.1.255
        gateway 192.168.1.1
        bridge_ports eth0 tap0
        bridge_fd 9
        bridge_hello 2
        bridge_maxage 12
        bridge_stp off
{% endhighlight %}

The next step is unbinding all ipv4 addresses from the eth0 interface and
restarting networking through systemd.

WARNING: this is the point of no return, there is a good chance you are
going to lock yourself out of the box. If you have no way of getting
a shell other than through the network, think twice before proceeding.
I take no responsibility of you break something. More information
[here](http://www.tldp.org/HOWTO/Ethernet-Bridge-netfilter-HOWTO-3.html).
{% highlight bash %}
# ifconfig eth0 0.0.0.0 down; systemctl restart networking
{% endhighlight %}

Or the non systemd way:
{% highlight bash %}
# ifconfig eth0 0.0.0.0 down; service networking restart
{% endhighlight %}
