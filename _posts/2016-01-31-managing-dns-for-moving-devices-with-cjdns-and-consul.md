---
title: Managing DNS for moving devices with cjdns and consul
layout: post
---
Over the past couple of years I've built up quite the collection of
re-purposed old laptops and desktops slowly contributing to global warming
from a stuffy closet at my parents' house and under the bed in my studio.
Even though my home lab is relatively small, it has introduced me to some
of the problems that can occur when dealing with running services in
different networks. I found that if I move around a lot and do my
computing on the go (trains, coffee places, my parents' place) some of the
more nuanced intricacies of multi-zone networking become apparent.

This post is about some specific use cases that I keep running into which
relate to the dynamics of having servers in different networks and
defining static addresses that can be used to reach those servers or have
those servers reach you regardless of where you are. Basically the
situation is like this: if I have servers a and b in network 1 and server
c in network 2, of which only b and c are publicly accessible, and I move
with my laptop between network 1 and 2 or even outside of both networks,
then there will be different paths that can be used to access each server
based on where the laptop is in relation to the networks of all other
servers. The problem is that these routes have to be manually reconfigured
every time the laptop or any of the nodes in the network changes its
relative position to any of the other nodes, which can be cumbersome and
unnecessary.

To show how moving devices can cause issues because the routes have to be
reconfigured every time they change, I present three cases. The first one
has to do with how you can get from a laptop to a not publicly accessible
server in another network by hopping though one that is, the second one is
about why using a reverse SSH tunnel is not the best solution when trying
to access a server behind a firewall and the third one is why a regular
VPN is not a panacea for this problem because they introduce a single
point of failure and they don't give you *one* endpoint that can be
reached from all nodes in the network if those nodes require an
intermediate hop.

Finally I will explain how mesh networking and distributed service
discovery can be used to solve these kind of problems. The goal of this
post is to demonstrate how to set up an environment in which as long as
you can make at least one connection to a server that knows how to reach
the destination you are trying to access you can just run

{% highlight bash %}
λ ~ consul members   # on server_c which can not reach server_a directly
Node          Address                    Status  Type    Build  Protocol  DC
server_a      [2001:0db8:...:0001]:8301  alive   server  0.6.0  2         cjdns
server_b      [2001:0db8:...:0002]:8301  alive   server  0.5.0  2         cjdns
server_c      [2001:0db8:...:0003]:8301  alive   server  0.6.3  2         cjdns
λ ~ SSH server_a.node.consul
Last login: Sun Jan 31 17:46:51 2016 from 2001:0db8:...:0003
{% endhighlight %}


and let the software figure out the route for you.

Case 1. Relative proxying
------------------------- 
This part attempts to make clear why there is no way to define static
hosts in an SSH configuration that will allow us to reach all three
servers from a laptop regardless of where it is in relation to the network
of the destination server when there is at least one possible route to
that machine. When I refer to the relation of the source machine to the
network of the destination machine I'm talking about whether or not
a connection can be made through a local network, a public network like
the internet, or if the server can be reached directly at all.

Depending on whether the laptop is in network 1, 2 or somewhere else,
there are different paths that can be travelled to access each server. The
paths diverge according to in which network the laptop resides. The laptop
can reach a server in the same network by using the internal network,
a path that will not be available anymore when the laptop changes
networks. But as long as there is at least one publicly accessible server
in each network all servers can be reached from anywhere using the same
route. 

For example, if the laptop is in network 1, it could reach both server
A and server b through the internal network but it would have to go over
the internet to reach C. In order to be able to access all servers from
the laptop when the laptop is in network 1 these SSH config rules could be
defined.

{% highlight python %}
Host server_a
    HostName 192.168.1.2    # server a in network 1

Host server_b
    HostName 192.168.1.3    # server b in network 1

Host server_c
    HostName c.example.com  # server c in network 2
{% endhighlight %}

The connections to all servers would then look like this

{% highlight bash %}
.========.   .===.
| laptop |->-| a |
'========'   '==='
.========.   .===.
| laptop |->-| b |
'========'   '==='
.========.   .----------.   .===.
| laptop |->-| internet |->-| c |
'========'   '----------'   '==='
{% endhighlight %}


But because server b is publicly accessible the laptop could also reach it
over the internet in the same way server c is accessed even though server
B and the laptop reside in the same network.

{% highlight python %}
Host server_b
    HostName b.example.com  # server b in network 1
{% endhighlight %}

Connecting to b would then work like this
{% highlight bash %}
.========.   .----------.   .===.
| laptop |->-| internet |->-| b |
'========'   '----------'   '==='
{% endhighlight %}

With this configuration server b and c are all set to be accessible even
when the laptop is moved to network 2 or even outside of 1 and 2 (say
a public access point). If server a needs to be reachable from anywhere
outside of network 1, it needs to be defined in terms of server b.

{% highlight python %}
Host server_a
    HostName 192.168.1.2    # server a in network 1
    ProxyCommand SSH -W %h:%p server_b
{% endhighlight %}

The laptop goes through the internet to get to server b, and server b goes
through the local network to get to server a.

{% highlight bash %}
.========.   .----------.   .===.   .===.
| laptop |->-| internet |->-| b |->-| a |
'========'   '----------'   '==='   '==='
{% endhighlight %}

When the laptop is not in network 1, going through b which can be accessed
externally is the only path there is to server a. But if the source is in
network 1, it could not only access server a like that but it could also
reach it through the internal network directly, or even go through the
internal network to server b and connect to server a from there.

{% highlight bash %}
.========.   .===.
| laptop |->-| a |
'========'   '==='
.========.   .===.   .===.
| laptop |->-| b |->-| a |
'========'   '==='   '==='
{% endhighlight %}

This is where the situation is encountered where the SSH configuration
does not work anymore while there still is a path to the destination
machine from the position of the laptop. If network 1 experiences an
internet outage, or if the DNS to server b changes, or if server b goes
offline, server a and b won't be accessible anymore using the static SSH
rules even when the laptop is in the same network. If the source can't
access b over the internet anymore, the SSH configuration has to be
updated back to the internal address in order to get to a even though
nothing happened to a itself.

{% highlight bash %}
.========.   .----------.   .===.   .===.
| laptop |->-| internet |-/-| b |->-| a |
'========'   '----------'   '==='   '==='
{% endhighlight %}

The laptop can still reach a by using the internal network.

{% highlight bash %}
.========.   .===.
| laptop |->-| a |
'========'   '==='
{% endhighlight %}

In this scenario there is no way around altering the SSH configuration
rules. Having to do so is inconvenient because as soon as the laptop
leaves network 1 it still won't be able to reach server b or a once the
outage is resolved or the DNS is fixed without changing the SSH config
rules to bounce through the public address of server b again.

So when the laptop with the updated SSH configuration moves outside of
network 1, server a won't be accessible anymore because it now connects
directly to the internal address.

{% highlight bash %}
.========.   .===.
| laptop |-/-| a |
'========'   '==='
{% endhighlight %}

This is a problem that can be solved with cjdns because it provides an
IPv6 address for all devices which will never change even if the device is
moved. Before explaining how that works, let's first look at a second
issue which is similar but slightly different.


Case 2. Impracticalities of reverse tunneling
---------------------------------------------
I've been entertaining the idea of running some services on the Linux
chroot on my phone for a while now but there are some obvious
impracticalities that come to mind. One of the major ones is that even
SSH-ing into the device can be a chore because I don't want to bind the
SSH server on rmnet0 (the 3G interface) which means that the device is not
always publicly accessible over the internet even when there *is* an
outbound connection. Additionally, because the device is always on the
move, there is also no way to ensure forwarding to the wlan interface on
the router level because most of the times you simply don't have the power
to change those settings.

Regardless of internet connectivity, how to access the same phone from the
same laptop through an internal network will change based on where you
are. In my particular situation setting a static address on the phone is
not a solution because I configured different subnets in my two wifi
networks to some avoid unrelated VPN collision issues, so the internal IP
addresses can't even be the same across networks. With most networks you
can't assume that you can retain an internal IP address when you move
devices to another one.

So if a device is not publicly accessible and there also are no other
publicly accessible nodes in the network that can be used to bounce
through, either because they don't exist or because the IP addresses in
the internal network are subject to change, then the only way is to resort
to reverse tunneling in order to set up an SSH connection from the
outside.

A reverse SSH tunnel is exactly what it sounds like: you take a port from
the source host and bind that on an interface on the destination host.
That forwarded port on the destination host can now be used to access the
port on the source. You can do this from behind a firewall because only
outbound connections are used.

The following command establishes a persistent reverse SSH tunnel from the
moving device to server b. The -g flag allows for remote hosts to access
the forwarded port, if that is omitted only local connections will be
forwarded.

{% highlight bash %}
autoSSH -M 0 -N -R 9922:localhost:22 server_b
{% endhighlight %}

Server a could now access the phone like this.

{% highlight bash %}
Host phone
    HostName 192.168.1.3    # server b in network 1
    Port 9922
{% endhighlight %}

Sever a connects to server b where the phone has forwarded its SSH port
to.

{% highlight bash %}
.===.   .===.   .----------.   .=======. 
| a |->-| b |-<-| internet |-<-| phone |
'==='   '==='   '----------'   '=======' 
{% endhighlight %}


But what if server c in network 2 wants to access the phone? It would
first need to hop to server b in network 1 and then SSH to the laptop in
the third network, which is very clunky.

{% highlight python %}
Host server_b
    HostName b.example.com  # server b in network 1

Host phone
    HostName localhost
    Port 9922
    ProxyCommand SSH -W %h:%p server_b
{% endhighlight %}

Server c now needs to go through the internet to get to server b, after
which server b will go through the internet again to get to the phone.

{% highlight bash %}
.===.   .----------.   .===.   .----------.   .=======. 
| c |->-| internet |->-| b |-<-| internet |-<-| phone |
'==='   '----------'   '==='   '----------'   '=======' 
{% endhighlight %}

This is a problem because now the connection are bound by the network
limits of server B. If server c and the phone have a better network
connection than server B then time is wasted by transferring it all
through B. 

Another issue is that when server b goes offline, server c might still be
publicly accessible. If that happens, there is still one publicly
available endpoint that can be used to forward the SSH port to, so there
is still a possible connection that can be made from the phone to make
it's SSH server accessible. However binding the port to server c instead
of b would cause the exact same issue from the perspective of the other
side. 

So should tunnels be set up preventively to all of the public servers in
all networks? For the amount of nodes in this example that might be
feasible, but if the network grows the complexity will increase and that
will result in an exponential amount of connections. And consequently if
there were tunnels from all nodes to all other nodes in the network there
would still be no way to pick an address that can be used in the SSH
configuration for all other nodes, which results in the same problems as
described in case 1.


Case 3. A single point of failure
---------------------------------
Most of the issues from the previous cases stem from the difference
between how one can get from one point to the other based on where those
points are in relation to each other. The majority of those problems could
be solved if there was a way to make all nodes perceive all others as if
they were in the same network. A traditional virtual private network can
do just that. 

All nodes would be able to reach all other nodes as if they were in the
same network and that would make defining a static SSH config pretty
straightforward.

{% highlight python %}
Host server_a
    HostName 10.0.0.1       # server a in network 1

Host server_b
    HostName 10.0.0.2       # server b in network 1

Host server_c
    HostName 10.0.0.3       # server c in network 2
{% endhighlight %}

But looking more closely it becomes clear that this just moves the problem
and doesn't really solve anything. There would still need to be one
publicly accessible VPN server that all nodes can access. If that server
goes down or becomes unreachable, the virtual network would go down as
well, making it a single point of failure. That specific scenario could
somewhat be remedied by some sort of automatic failover mechanism but that
brings all the issues from case 2 back around because you would still have
situations where you can't reach the same address from all devices.

One of the scenarios that I haven't alluded to yet is that of what happens
when one of the servers that could previously access another publicly
accessible server in another network stops being able to make outbound
connections over the internet but can still access other servers
internally.

If the VPN is running on publicly accessible server c, and server a can
reach server b, but server a can't reach the internet anymore, the route
from server a to the VPN would have to be reconfigured by making it go
through b. This is similar to server c having to go through b in order to
get to a instead of directly connecting to a as described in one of the
previous examples but the difference is that now a conventional port
forward can be used.

{% highlight bash %}
.===.   .----------.
| a |-/-| internet |
'==='   '----------'
.===.   .===.   .----------.   .===. 
| a |->-| b |->-| internet |->-| c |
'==='   '==='   '----------'   '===' 
{% endhighlight %}

The VPN port can be forwarded on server c to server b so that a can access
it as well and all nodes can appear as if they were in the same network
again even though they can't all reach the VPN server directly.

{% highlight bash %}
autoSSH -M 0 -N -L 9194:localhost:1194 server_c -g 
{% endhighlight %}

But this results in the same problems as before. What happens if this
scenario presents itself in other zones as well? Should all nodes forward
the VPN port to itself so that nodes can reach the VPN from wherever they
are by just using the nearest tunnel? There would still be no *one*
address or forwarded port that could be defined for the VPN that can be
accessed from all nodes that have at least one possible route.


Solution: a mesh overlay network
--------------------------------
It appears that what would really solve this problem is a decentralized
way to set up something that resembles a virtual private network. All
nodes would have to be able to connect to the network directly by
connecting to a node that is connected to the network already and that way
no manual hops would be required because that would all be abstracted away
to the networking layer. This would allow us to reach to all connected
nodes in the network using the same address without worrying about where
they are in relation to each other.

Luckily something exactly like that exists.
[Cjdns](https://github.com/cjdelisle/cjdns) is a project that originated
from the [/r/darknetplan](reddit.com/r/darknetplan) subreddit, the idea is
that it provides an encrypted IPv6 network that uses distributed hash
tables for routing with the goal of establishing a decentralized network
that provides unintrusive and ubiquitous security.  In simpler terms: it
enables you to run some cables to your neighbors and set up your own
internet without using centralized authorities. Even though your left and
right neighbor won't be able to connect to each other directly (because
they have to go through you), it will appear as if they can because the
routing protocol will figure out the path for you.

cjdns is different from similar projects like I2P and Tor because the goal
is not anonymity but secure and refractorized networking. The biggest
advantage for this use-case is that cjdns operates on level 3 of the
networking stack. Contrary to I2P, you don't have to adapt your
application to work with the protocol. cjdns simply gives you an IPv6
interface and takes care of the rest, this means that any application that
supports IPv6 can run on top of cjdns.

Once cjdns is set up, one static SSH configuration can be defined that
will work in all situations as long as there is at least one route to the
destination you are trying to reach. There is no need to worry about the
hops anymore because cjdns will take care of this. 

{% highlight python %}
Host server_a
    # cjdns IPv6 address of server a
    HostName 2001:0db8:...:0001  

Host server_b
    # cjdns IPv6 address of server b
    HostName 2001:0db8:...:0002  

Host server_c
    # cjdns IPv6 address of server c
    HostName 2001:0db8:...:0003  
{% endhighlight %}

This solves the problems described in the three cases but it depends on
one condition: all nodes would have to have to be peered to all other
nodes if the goal is complete decentralization. That way when the laptop
moves from network 1 to network 2 it will not matter that it loses the
internal route to the servers in network 1 because it will gain internal
routes to all servers in network 2. The public routes will still exist but
if they go down (for example if the network you are in loses internet
connectivity) you will still be able to make connections through the
routes that exist in the local network because you are peered internally
as well.

cjdns also offers a public network that you can connect to. This could be
an option to consider if you don't want to maintain your own publicly
accessible endpoints. There are still points of failures using cjdns
across zones but now they are limited to the amount of nodes in a network
that can reach a peer that can be accessed by other machines in other
networks. The public network is called Hyperboria. Hyperboria is pretty
[impressive](http://www.fc00.org/) and it is certainly a beacon of hope in
the current declining free internet of walled gardens and [The War On
General Purpose
Computing](http://boingboing.net/2012/01/10/lockdown.html), but for my use
case I decided to roll my own private cjdns network, partially because
I do not feel comfortable relaying other people's possibly illicit traffic
through my internet connections but also because for my sporadic tinkering
I didn't feel like it was worth it yet to lock down the security on my
personal servers with strict firewalls to make them secure enough to use
in a public IPv6 network.

Adding DNS into the mix
--------------------------
Now that there are IPv6 addresses that can be used to access any device in
the network as long as there is at least one route available to that
device, either directly or through one or more other machines, it would be
nice if there was a way to assign a human-friendly address to it. 

As long as this setup is only used for SSH this isn't a very important
consideration because it doesn't really matter if you define an IP address
or a domain name as hostname in the configuration because after doing it
once you can forget about it. But you could take it one step further if
you wanted to use this configuration for more than just remote access to
moving devices over SSH. You could now also access other services on all
nodes from anywhere as long as there is at least one route that can be
taken and the service is bound to the tun0 interface.

The crazy scientist in me gets really excited by the idea of running
a web-server on a mobile phone in my pocket, which would be a stupid idea
for many reasons but a great idea for many others, especially if there was
a way to combine it with some form of distributed load balancing. For now
let's focus on a situation where we need to host a service on one of the
nodes in the network and that service needs to be accessible from all
other connected nodes using a specific domain name.

There is probably something that could be done with dynamic DNS now that
all the nodes are in the same (virtual) network, but since this
configuration doesn't use any centralized dependencies up until this point
it would be nice if the hostname resolution could be set up with this
constraint as well. The first thing that comes to mind is
[Namecoin](https://namecoin.info/), but for this particular use-case there
is a possibly more fitting solution: [consul by
Hashicorp](https://www.consul.io/). 

Consul offers a query interface for DNS. You can post new entries to its
API or define them in configuration files that are read by the agent.
These entries will then be propagated to all other nodes in the quorum.
Because consul provides a DNS server on the localhost of each node that
runs the agent, you can just set up a dnsmasq rule that points to consul's
DNS port (8600 by default) and that will be enough to give you access to
the \*.consul domain.

Out of the box all active nodes will be available on the \*.nodes.consul
subdomain. If one of your hosts is named *server_a* all nodes in the
consul cluster will be able to reach it on *server_a.node.consul* as
long as they have the dnsmasq configured.

{% highlight python %}
$ echo "server=/consul./127.0.0.1#8600" > /etc/dnsmasq.d/10-consul
{% endhighlight %}

The advantage consul has over Namecoin for this particular situation is
that it is trivial to completely self-host it and it doesn't involve
a public blockchain. There is one big downside to using consul though, and
that is that because it uses the [Raft consensus
algorithm](https://en.wikipedia.org/wiki/Raft_\(computer_science\)) it
requires at least three active servers to form a quorum. If there are less
than three nodes, consul can't perform the leader election procedure and
there is no way to reliably distribute the information throughout the
cluster.

Once consul is set up you could perform a dig the domain on all nodes and this is
what you would see:
{% highlight python %}
$ dig server_a.node.consul ANY

; <<>> DiG 9.9.5-9+deb8u5-Debian <<>> server_a.node.consul ANY
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 18639
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;server_a.node.consul.      IN      ANY

;; ANSWER SECTION:
server_a.node.consul. 0     IN      AAAA
2001:0db8:...:0001

;; Query time: 81 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Sun Feb 07 19:44:34 CET 2016
;; MSG SIZE  rcvd: 94
{% endhighlight %}



Next time: bringing it all to life
-----------------------
So far I only discussed the theoretical aspects of this contraption,
unfortunately this post ended up a lot longer than I expected so for now
I'm going to cut it short. Tune in soon for part two of this post where
I'll describe step by step how to get this all set up. If you want to give
it a try yourself (this really isn't rocket science), check out the
repositories for [cjdns](https://github.com/cjdelisle/cjdns) and
[consul](https://github.com/hashicorp/consul) on github. Getting cjdns up
and running is a piece of cake and the
[documentation](https://www.consul.io/intro/) for consul is really good so
if you want to give it a try on your own you should be able to figure it
out.
