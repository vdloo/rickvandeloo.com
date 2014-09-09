---
title: `Not authorized to use this service` in DS Audio and Video 
layout: post
---
Since I updated my phone to iOS7 I've been encountering the following error when attempting to log in using the DS Audio and Video apps.
<br>
<br>

-> _"you are not authorized to use this service"_ <-

<br>
Reinstalling the apps didn't solve the issue for me and the error seemed exclusive to my phone as logging in still worked flawlessly on other devices running the same version of the OS.    

Since I didn't want to resort to resetting my phone, I did some more troubleshooting and these are the solutions (and possible problems causing the error) I came across, so if you are experiencing the same error, trying these things might save you a headache.	

<br>

###Checking the automatic block-list	

To check if this problem is causing your error, change the ip address of your iOS device and attempt to log in. If the error no longer occurs, there is a good chance that the Synology NAS has blocked your ip address. 

To make sure this was the cause of the problem, and to possibly prevent future similar things from happening do the following:


In the DSM control panel, click on the icon of Auto Block and see if you have this option enabled. Also check the Block List tab to see if some (or any) of the ips listed there are those of your devices. 

If you have shell access to the NAS you can also look in the files /etc/host.deny and /etc/hosts.allow to see if there is anything interesting there that might prevent you from logging in. 
<br><br>

###Updating the DSM time settings   

If the previous bullet point didn't solve your problem, you might try updating the time settings of your DSM. Do this in Control Panel -> Regional Options -> Time. Try changing the Time Setting from 'Manually' to 'Synchronize with a NTP server' and press the 'Update Now' button. An offset of a couple of minutes could be enough to cause the 'you are not authorized to use this service' error. To achieve the same effect from the commandline, you could also run the following command: 		

{% highlight bash %}
# ntpd -q
{% endhighlight %}   

<br>

###Changing the iOS time setting from '24-Hour Time' to '12-Hour Time'   

This step is what I think finally solved the problem for me. For some reason the combination of turning off 24 Hour time (and restarting the phone) was what enabled me to log into the Audio Station using the DS Audio app once again. I have no idea if this was what actually caused the problem, and I couldn't reproduce the problem afterwards either, but if the previous approaches were of no help, you could give it a shot. 

If these three things didn't solve the problem for you, you could try to enable logging in the Audio Station -> Settings -> Options menu to take a look under the hood, and maybe this log in combination with the /var/log/synolog file will give you some insights on what might be the cause of your problem. 
