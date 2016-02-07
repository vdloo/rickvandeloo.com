---
title: Managing dns for moving devices with cjdns and consul
layout: post
---
Over the past couple of years I've built up quite the collection of
re-purposed old laptops and desktops slowly contributing to global warming
from a stuffy closet at my parents' house and under the bed in my studio.
I like to pretend that my total apathetic disregard for the rainforests
comes from a place of utility but in practice it probably has to do more
with my obsessive need for hoarding places to SSH into, something
I attribute to playing too much Mega Man Battle Network in high-school.

Nevertheless, I have some specific use cases I keep running into and it
involves these make-shift servers and me not wanting to have to remember
how to reach them from my laptop or phone when I move between my place, my
parents' place or anywhere else in the world.

Basically the situation is like this: if I have servers A and B in network
1 and server C in network 2, of which only B and C are publicly
accessible, and I move with my laptop between network 1 and 2 or even
outside of both networks, then there will be different paths that can be
used to access each server based on where the laptop is in relation to the
networks of all other servers and these routes have to be reconfigured
each time the laptop or any other node in the network changes it's
relative position to any other nodes which is cumbersome and unnecessary.

To illustrate how moving machines can be problematic because they require
constant reconfiguration of the routes to other machines I present three
cases. The first one has to do with how we get from a laptop to a not
publicly accessible server in another network by hopping though one that
is, the second one is about why using a reverse SSH tunnel is not the best
solution when trying to access a server behind a firewall and the third
one is why a regular VPN is not a panacea for moving machines because they
introduce a single point of failure and don't give you *one* endpoint that
can be reached from all nodes in a network if those nodes require an
intermediate hop.

Finally I will explain how to leverage mesh networking and distributed
service discovery by setting up cjdns and consul so that as long as you
can make at least one connection to a server that knows how to reach the
destination you are trying to access you can just run

{% highlight bash %}
λ ~ consul members   # on server_C which can not reach server_A directly
Node          Address                    Status  Type    Build  Protocol  DC
server_A      [2001:0db8:...:0001]:8301  alive   server  0.6.0  2         cjdns
server_B      [2001:0db8:...:0002]:8301  alive   server  0.5.0  2         cjdns
server_C      [2001:0db8:...:0003]:8301  alive   server  0.6.3  2         cjdns
λ ~ SSH server_A.node.consul
Last login: Sun Jan 31 17:46:51 2016 from 2001:0db8:...:0003
{% endhighlight %}


and let the software figure out the route for you.

Case 1. Relative proxying
-------------------------
This section demonstrates why there is no way to define static hosts in an
SSH configuration that will allow us to reach all three servers from
a laptop regardless of where it is in relation to the network of the
destination server when there is at least one possible route to that
machine. When I refer to the relation of the source machine to the network
of the destination machine I'm talking about whether or not we can make
a connection through a local network, a public network like the internet,
or if we can reach the server directly at all.

Depending on whether the laptop is in network 1, 2 or somewhere else,
there are different paths that can be travelled to access each server. The
paths differ according to in which network the laptop resides. The laptop
can reach a server in the same network by using the internal network,
a path that will not be available anymore when the laptop changes
networks. But as long as there is at least one publicly accessible server
in each network all servers can be reached from anywhere using the same
route. 

For example, if the laptop is in network 1, it could reach both server
A and server B through the internal network but it would have to go over
the internet to reach C. In order to be able to access all servers from
the laptop when the laptop is in network 1 these SSH config rules could be
defined.

{% highlight python %}
Host server_A
    HostName 192.168.1.2    # server A in network 1

Host server_B
    HostName 192.168.1.3    # server B in network 1

Host server_C
    HostName c.example.com  # server C in network 2
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


But because server B is publicly accessible the laptop could also reach it
over the internet in the same way server C is accessed even though server
B and the laptop reside in the same network.

{% highlight python %}
Host server_B
    HostName b.example.com  # server B in network 1
{% endhighlight %}

Connecting to B would then work like this
{% highlight bash %}
.========.   .----------.   .===.
| laptop |->-| internet |->-| b |
'========'   '----------'   '==='
{% endhighlight %}

With this configuration server B and C are all set to be accessible even
when we move the laptop to network 2 or even outside 1 and 2 (say a public
access point). If we also want to be able to reach server A from anywhere
outside of network 1, we need to define it in terms of server B. We can do
this by using server B as a proxy. If we can access server B, and server
B can access server A, then we can reach A which is not publicly
accessible from outside the network by going through B which *is* publicly
accessible.

{% highlight python %}
Host server_A
    HostName 192.168.1.2    # server A in network 1
    ProxyCommand SSH -W %h:%p server_B
{% endhighlight %}

We go through the internet to get to server B, and then we go from server
B to get to server A.
{% highlight bash %}
.========.   .----------.   .===.   .===.
| laptop |->-| internet |->-| b |->-| a |
'========'   '----------'   '==='   '==='
{% endhighlight %}

When the laptop is not in network 1, going through B which can be accessed
externally is the only path we have to server A. But if we are in network
1, we could not only access server A like that but we could also reach it
through the internal network directly, or even go through the internal
network to server B and connect to server A from there.
{% highlight bash %}
.========.   .===.
| laptop |->-| a |
'========'   '==='
.========.   .===.   .===.
| laptop |->-| b |->-| a |
'========'   '==='   '==='
{% endhighlight %}

This is where we encounter the situation where the SSH configuration does
not work anymore while there still is a path to the destination machine
from the position of the laptop. If network 1 experiences an internet
outage, or if the DNS to server B changes, or if server B goes offline,
server A and B won't be accessible anymore using the static SSH rules even
when the laptop is in the same network. If we can't access B over the
internet anymore, we have to manually update our SSH configuration back to
the specific internal addresses in order to get to A even though nothing
happened to A.
{% highlight bash %}
.========.   .----------.   .===.   .===.
| laptop |->-| internet |-/-| b |->-| a |
'========'   '----------'   '==='   '==='
{% endhighlight %}

We can still get to A by using the internal network.
{% highlight bash %}
.========.   .===.
| laptop |->-| a |
'========'   '==='
{% endhighlight %}

We are forced to alter the SSH config rules back to the internal
configuration, which we don't want to do because as soon as the laptop
leaves network 1 we still won't be able to reach server B or A once the
outage is resolved or the DNS is fixed without changing the SSH config
rules again.

So when we move outside of network 1 with our updated SSH configuration,
server A won't be accessible anymore because it now connects directly to
the internal address.
{% highlight bash %}
.========.   .===.
| laptop |-/-| a |
'========'   '==='
{% endhighlight %}

<br>

Case 2. Impracticalities of reverse tunneling
---------------------------------------------
I've been entertaining the idea of running some services on the Linux
chroot on my phone for a while now but there are some obvious
impracticalities that come to mind. One of the major ones is that even
SSH-ing into the device can be a chore because I don't want to bind the
SSH server on rmnet0 (the 3G interface) which means that the device is not
always publicly accessible over the internet even when there *is* an
outbound connnection. Additionally, because the device is always on the
move, there is also no way to ensure forwarding to the wlan interface on
the router level because most of the times you simply don't have the power
to change those settings.

Regardless of internet connectivity, how to access the same phone from the
same laptop through an internal network will change based on where you
are. In my particular situation setting a static address on the phone is
not a solution because I configured different subnets in my two wifi
networks to some avoid unrelated VPN collision issues, so the internal IP
addresses can't even be the same across networks. Point being: you can't
assume that you can retain an internal IP address when you change
networks.

So if a device is not publicly accessible and there also are no other
publicly accessible nodes in the network that we can use to bounce
through, either because they don't exist or because the IP addresses in
the internal network are subject to change, then we need to resort to
reverse tunneling in order to set up an SSH connection from the outside.

A reverse SSH tunnel is exactly what it sounds like: you take a port from
the source host and bind that on an interface on the destination host.
That forwarded port on the destination host can now be used to access the
port on the source. You can do this from behind a firewall because we are
only using outbound traffic.

The following command establishes a persistent reverse SSH tunnel from the
moving device to server B. The -g flag allows for remote hosts to access
the forwarded port, if we omit that only local connections will be
forwarded.
{% highlight bash %}
autoSSH -M 0 -N -R 9922:localhost:22 server_B
{% endhighlight %}

Server A could now access the phone like this.

{% highlight python %}
Host phone
    HostName 192.168.1.3    # server B in network 1
    Port 9922
{% endhighlight %}

Sever A connects to server B where the phone has forwarded its SSH port
to.
{% highlight bash %}
.===.   .===.   .----------.   .=======. 
| a |->-| b |-<-| internet |-<-| phone |
'==='   '==='   '----------'   '=======' 
{% endhighlight %}


But what if server C in network 2 wants to access the phone? It would
first need to hop to server B in network 1 and then SSH to the laptop in
the third network, which is very clunky.

{% highlight python %}
Host server_B
    HostName b.example.com  # server B in network 1

Host phone
    HostName localhost
    Port 9922
    ProxyCommand SSH -W %h:%p server_B
{% endhighlight %}

Server C now needs to go through the internet to get to server B, after
which we will go through the internet again to get to the phone.
{% highlight bash %}
.===.   .----------.   .===.   .----------.   .=======. 
| c |->-| internet |->-| b |-<-| internet |-<-| phone |
'==='   '----------'   '==='   '----------'   '=======' 
{% endhighlight %}

This is a problem because now we are bound by the network limits of server
B. If server C and the phone have a better network connection than server
B then we are wasting time by transferring it all through B. 

Another issue is that when server B goes offline, server C might still be
publicly accessible. If that happens, there is still one publicly
available endpoint that can be used to forward the SSH port to, so there
is still a possible connection that can be made from the phone to make
it's SSH server accessible. However binding the port to server C instead
of B would cause the exact same issue from the perspective of the other
side. 

So should we preventively set up tunnels to all of our public servers in
all networks? For the amount of nodes in this example that might be
feasible, but if the network grows the complexity will increase and we
will end up with a Cartesian product. Furthermore, if we did set up
a tunnel to all nodes in the network, we'd still have to pick the address
to use in the SSH configuration of all other nodes, which results in the
same problems from case 1.


Case 3. A single point of failure
---------------------------------
Most of the issues from the previous cases stem from the difference
between how we can get from one point to the other based on where those
points are in relation to each other. The majority of those problems could
be solved if we had a way to make all nodes perceive all others as if they
were in the same network. A traditional virtual private network can do
just that. 

All nodes would be able to reach all other nodes as if they were in the
same network and that would make defining a static SSH config pretty
straightforward.

{% highlight python %}
Host server_A
    HostName 10.0.0.1       # server A in network 1

Host server_B
    HostName 10.0.0.2       # server B in network 1

Host server_C
    HostName 10.0.0.3       # server C in network 2
{% endhighlight %}

But looking more closely we see that this just moves the problem and
doesn't really solve anything. We'd still have to have one publicly
accessible VPN server that all nodes can access. If that server goes down
or becomes unreachable, the virtual network would go down as well, making
it a single point of failure. That specific scenario could somewhat be
remedied by some sort of automatic failover mechanism but that brings all
the issues from case 2 back around because you would still have situations
where you can't reach the same address from all devices.

One of the scenarios that we haven't discussed yet is that of what happens
when one of the servers thats could previously access another publicly
accessible server in another network stops being able to make outbound
connections over the internet but can still access other servers
internally.

If we run the VPN on publicly accessible server C, and server A can reach
server B, but server A can't reach the internet anymore, we'd have to
reconfigure the route to the VPN from server A by making it go through B.
This is similar to server C having to go through B in order to get to
A instead of directly connecting to publicly inaccessible server A as
described in one of the previous examples but the difference is that now
we can solve it with a conventional port forward.

{% highlight bash %}
.===.   .----------.
| a |-/-| internet |
'==='   '----------'
.===.   .===.   .----------.   .===. 
| a |->-| b |->-| internet |->-| c |
'==='   '==='   '----------'   '===' 
{% endhighlight %}

We can forward the VPN port on server C to server B so that A can access
it and all nodes can appear as if they were in the same network again even
though they can't all reach the VPN server directly.

{% highlight bash %}
autoSSH -M 0 -N -L 9194:localhost:1194 server_C -g 
{% endhighlight %}

But this results in the same problems as we've seen before. What happens
if this scenario presents itself in other zones as well? Do we make all
nodes forward the VPN port to itself so that we can reach the VPN from
wherever we are by just using the nearest tunnel? There would still be no
*one* address or forwarded port that could be defined for the VPN that can
be accessed from all nodes that have at least one possible route that they
could take.


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

Cjdns is different from similar projects like I2P and Tor because the goal
is not anonymity but secure and refractorized networking. The biggest
advantage for this use-case is that cjdns operates on level 3 of the
networking stack. Contrary to I2P, you don't have to adapt your
application to work with the protocol. Cjdns simply gives you an IPv6
interface and takes care of the rest, this means that any application that
supports IPv6 can run on top of cjdns.

If we get cjdns set up, we could define one static SSH configuration that
will work in all situations as long as there is at least one route to the
destination you are trying to reach. We don't need to worry about the hops
anymore because cjdns will take care of this. 

{% highlight python %}
Host server_A
    # cjdns IPv6 address of server A
    HostName 2001:0db8:...:0001  

Host server_B
    # cjdns IPv6 address of server B
    HostName 2001:0db8:...:0002  

Host server_C
    # cjdns IPv6 address of server C
    HostName 2001:0db8:...:0003  
{% endhighlight %}

This solves the problems described in the three cases but it depends on
one condition: all nodes would have to have to be peered to all other
nodes if we want to have complete decentralization. That way when the
laptop moves from network 1 to network 2 it will not matter that it loses
the internal route to the servers in network 1 because it will gain
internal routes to all servers in network 2. The public routes will still
exist but if they go down (like for example if the network you are in
loses internet connectivity) you will still be able to make connections
through the routes that exist in the local network because you are peered
internally as well.

Cjdns also offers a public network that you can connect to. This could be
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
Now that we have IPv6 addresses that can be used to access any device in
the network as long as there is at least one route available to that
device, either directly or through one or more other machines, it would be
nice if we could assign a human-friendly address to it. 

As long as we stick to only using this setup for SSH  this isn't a very
important consideration because it doesn't really matter if you define an
IP address or a domain name as hostname in the configuration because after
doing it once you can forget about it. But we could take it one step
further if we wanted to use this configuration for more than just remote
access to moving devices over SSH. We could now also access other services
on all nodes from anywhere as long as there is at least one route that can
be taken and the service is bound to the tun0 interface.

The crazy scientist in me gets really excited by the idea of running
a web-server on a mobile phone in my pocket, which would be a stupid idea
for many reasons but a great idea for many others, especially if there was
a way to combine it with some form of distributed load-balancing. For now
let's focus on a situation where we want to host and access a service on
one of the nodes in the network and that service needs to be accessible
from all other connected nodes using a specific domain name.

We could probably do something with dynamic DNS now that all the nodes are
in the same (virtual) network, but since we got this far without
introducing any centralized dependencies it would be nice if we also could
set up hostname resolution with this constraint as well. The first thing
that comes to mind is [Namecoin](https://namecoin.info/), but for this
particular use-case there is a possibly more fitting solution: [consul by
Hashicorp](https://www.consul.io/). 

Consul offers a query interface for DNS. You can post new entries to its
API or define them in configuration files that are read by the agent.
These entries will then be propagated to all other nodes in the quorum.
Because consul provides a DNS server on the localhost of each node that
runs the agent, you can just set up a dnsmasq rule that points to consul's
DNS port (8600 by default) and that will be enough to give you access to
the \*.consul domain.

Out of the box all active nodes will be available on the \*.nodes.consul
subdomain. If one of your hosts is named *server_A* all nodes in the
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
