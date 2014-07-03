---
title: Beacontalk - Peer to peer chat-program that sends data over Wi-Fi without associations
layout: post
---
###Beacontalk - Peer to peer chat-program that sends data over Wi-Fi without associations
Beacontalk is a peer to peer chat-program that uses a method of [sending data over Wi-Fi without associations](http://rickvandeloo.com/2012/08/01/Sending-data-over-Wi-Fi-without-associations/). This program enables the user to send text to other in-range wireless devices also running the program. All received messages that are unique, are re-broadcast. This means that if nodes are places in a straight line, the data will be sent through a chain of nodes. Be wary of that this program is merely a proof of concept and does not use any encryption. Anyone with a package-sniffer will be able to decipher your messages.

To install the program

{% highlight bash %}
$ ./configure; make; make install
{% endhighlight %}
Put your wireless card into monitor mode and pick a channel

{% highlight bash %}
$ ifconfig wlan0 down; iwconfig wlan0 mode monitor; iwconfig wlan0 channel 1; ifconfig wlan0 up
{% endhighlight %}
Run the program by doing

{% highlight bash %}
$ ./beacontalk wlan0
{% endhighlight %}
[Look at the source on github](https://github.com/vdloo/Beacontalk), or [download a tarball](https://github.com/vdloo/Beacontalk/zipball/master).
