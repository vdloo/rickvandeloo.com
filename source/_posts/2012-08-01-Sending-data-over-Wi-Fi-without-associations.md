---
title: Sending data over Wi-Fi without associations
layout: post
---
###Sending data over Wi-Fi without associations
For the past couple of months I’ve been on and off working on a project I call [sailingstone](http://i.imgur.com/YFyPY.png), although that particular project probably won’t see the light of day for a long time (if ever) it did point me in a direction that I found remarkably interesting. I needed to find a way to transmit data from one wireless card to another, without a router in between. There already exist various ad-hoc methods to achieve this, however most of them focus on a sole connection between two devices. For purposes such as mesh-networking it would be nice to be able to change connections swiftly or even broadcast to multiple nodes.

My first instinct was to just hurl packets into the air and have other nodes catch them. Imagine one laptop sending some bytes into the air containing a message, and another one grabbing those bytes. Unfortunately, that is not how things work. Wireless networking works by packing data into things called [frames](http://en.wikipedia.org/wiki/IEEE_802.11#Frames). This way a computer can very quickly decide if it has a use for an incoming packet by looking at the header. If that header does not meet the criteria of what your wireless card is configured for, it just ignores the packet and goes on to the next one.

You could modify how your wireless card acts by patching your drivers, but that wouldn’t be very practical due to incompatibility with other devices. Instead of doing that, we will utilize the existing infrastructure and add our data to a recognized frame. Using a tool called [mdk3](http://homepages.tu-darmstadt.de/~p_larbig/wlan/#mdk3) I generated a fake beacon frame for an encrypted access point with the SSID ‘stone’. This is where it gets a bit technical.

Let’s create two programs, the first one being a transmitter and the second one being a receiver:  

[transmitter.c](https://github.com/vdloo/Beacontalk/blob/master/extra/transmitter.c)
[receiver.c](https://github.com/vdloo/Beacontalk/blob/master/extra/receiver.c)

To compile these programs use the -lpcap flag:

{% highlight bash %}
$ gcc transmitter.c -o transmitter -lpcap
$ gcc receiver.c -o receiver -lpcap
{% endhighlight %}
In transmitter.c you can see how memcpy is used to first copy the frame header into the variable buf. This is interesting because you could use this method to capture any beacon frame, grab the header, and re-broadcast it with your added data. The crafted packet gets broadcast using the pcap_sendpacket function. 

{% highlight c %}
u_char buf[1024];
char message[850], messagelen[6];
char size[] = "size", end[] = "end";
char type[] = 	"\x0\x0\xd\x0\x4\x80\x2\x0\x2\x0\x0"
		"\x0\x0\x80\x0\x0\x0\xff\xff\xff\xff"
		"\xff\xff\xf8\xb7\x6f\x7b\xc2\x59\xf8"
		"\xb7\x6f\x7b\xc2\x59\x0\x0\x0\x0\x0"
		"\x0\x0\x0\x0\x0\x64\x0\x11\x0\x0\x5"
		"\x73\x74\x6f\x6e\x65\x1\x4\x82\x84"
		"\x8b\x96\x3\x1\xc\x4\x6\x1\x2\x0\x0"
		"\x0\x0\x5\x4\x0\x1\x0\x0\xdd\x18\x0"
		"\x50\xf2\x1\x1\x0\x0\x50\xf2\x4\x1"
		"\x0\x0\x50\xf2\x4\x1\x0\x0\x50\xf2"
		"\x2\x0\x0"; /* stone beacon frame header */
/* adds the beacon frame */ 
memcpy(buf, type, sizeof(type));

/* adds the size marker */
memcpy(buf+sizeof(type), size, strlen(size));

/*adds the size number */
sprintf(messagelen, "%.5d", strlen(message));
memcpy(buf+sizeof(type)+strlen(size), messagelen, 5);

/* adds the message */
memcpy(buf+sizeof(type)+strlen(size)+sizeof(messagelen), message, strlen(message));

/* adds end marker */	
memcpy(buf+sizeof(type)+strlen(size)+sizeof(messagelen)+strlen(message), end, strlen(end));
{% endhighlight %}

The receiving end should be pretty straightforward if you are familiar with the pcap library. In receiver.c, pcap_loop is used to capture incoming packets and gets them processed by using got_packet, which is a callback function. This means that every time a packet gets captured it is passed to that function as a parameter. This function then subjects the packet to the following lines of code to determine if the packet meets our filter’s demands (i.e. having ‘stone’ as the SSID and having a valid size marker). When everything is in order, it will print the message sent by the server character by character. 

{% highlight c %}
/* drop everything that does not contain "x0\x11\x0\x0\x5\x73\x74\x6F\x6E\x65" */
identified = 0;
for (i = 0; i < 300; i++){
        if (identified == 0 && packet[i] == 's' && packet[i+1] == 't' && packet[i+2] == 'o' && packet[i+3] == 'n' && packet[i+4] == 'e'){
                identified = 1; 
        }       
}       
if (identified){
        /* looks for beginning of datastring and its size */ 
        start = 0;
        intlength = 0;
        for (i = 1; i < 1024; i++){
                if (start == 0 && packet[i] == 's' && packet[i+1] == 'i' && packet[i+2] == 'z' && packet[i+3] == 'e'){
                        start = i;
                }       
        }       
        if (start > 0){
                for (i = 0; i < 5; i++){
                        charlength[i] = packet[start+i+4];
                }       
                intlength = atoi(charlength);
        }       

        /* print output */
        if (intlength > 0){
                for (i = start+9; i <= start+9+intlength; i++){
                        fprintf(stdout, "%c", packet[i]);
                }       
        }       
} 
{% endhighlight %}
To run the programs you must make sure of a couple of things. Firstly, make sure to have the proper permissions to use the wireless adapter. Secondly make sure that the wireless adapter(s) that you will be using is/are in monitor mode. And lastly that the transmitter and the receiver are using the same frequency (channel).

Running the program as root (or sudo) should take care of the permissions. If you are using the madwifi drivers you can do the following to get the wireless adapter into monitor mode. I am not sure on how to go about this with other drivers, but Google can probably help you with that. 

Firstly bring the adapter down using:

{% highlight bash %}
$ ifconfig wlan0 down
{% endhighlight %}
Then do this to put the device in monitor mode:

{% highlight bash %}
$ iwconfig wlan0 mode monitor
{% endhighlight %}
Where wlan0 is your wireless adapter. 

To change the channel of your device you can do:

{% highlight bash %}
$ iwconfig wlan0 channel 1
{% endhighlight %}
Or alternatively enter frequency manually:

{% highlight bash %}
$ iwconfig wlan0 freq 2.412G
{% endhighlight %}
Finally bring your adapter back online with:

{% highlight bash %}
$ ifconfig wlan0 up
{% endhighlight %}
If you did this for both the devices you are going to use, you should now be ready to successfully run the transmitter and receiver programs. After compiling the program you can run it by opening up two terminals, and running for example

{% highlight bash %}
$ ./transmitter wlan0
{% endhighlight %}
in one terminal, and

{% highlight bash %}
$ ./receiver wlan1
{% endhighlight %}
in the other. If you don’t have access to two wireless cards or you can’t be bothered to whip out another laptop or something you can also just run both programs using one adapter. This will however cause the receiver program to pick up and print all packets twice: once when it is broadcast and once when it is received. 

You can now enter text in the transmitter program and once you press enter, it will be sent. You can also cat a text file and pipe its output into the program, this will send the contents of that file line by line.

{% highlight bash %}
$ cat file.txt | ./transmitter wlan1
{% endhighlight %}
Remember, not all packets might reach their destination and show up on the receiving side. There is no error checking involved. Also, the data is not encrypted. Anyone can just grab these packets out of the air. You could however implement these features yourself. 

There are obvious advantages and disadvantages to using this method. An advantage might be how a technique like this could be implemented on any device that is capable of scanning for wireless networks. A disadvantage is that this would be very hard to scale.

As an example of what this can be used for, I created a program called [beacontalk](http://rickvandeloo.com/2012/08/01/Beacontalk-peer-to-peer-chat-program-that-sends-data-over-Wi-Fi-without-associations/). Beacontalk is a simple peer to peer chat-program that re-broadcasts all incoming messages. This means that if there are three nodes, and node A can reach node B, and B can reach C, but node A can not reach C, then the message will get passed along by B to still reach all the nodes in the network.

[You can get the source for beacontalk on github.](https://github.com/vdloo/Beacontalk)
