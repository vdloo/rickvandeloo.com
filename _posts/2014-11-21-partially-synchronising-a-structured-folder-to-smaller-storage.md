---
title: Partially synchronising a structured folder (on a Synology machine) to a smaller storage
layout: post
---
Partially synchronising a structured folder (on a Synology machine) to
a smaller storage

One of the most enjoyable things writing code has to offer is automating
tasks to the point where you can completely forget about them. There is
nothing like walking down the street and getting an email on your phone
from some long forgotten process, reminding you that something happened
and thinking “I can’t remember how that works but I built that”. Recently
I’ve been dusting off and cleaning up some of my personal scripts to
publish up on [my GitHub account](https://github.com/vdloo) and in the
process I stumbled upon a problem I solved a long time ago which on
initial glance seemed really simple but ended up being rather treacherous. 

The situation is as follows: imagine a Synology server storing a video
library where files are stored with a certain directory structure. This
structure is important because Kodi (previously XBMC) requires [specific
naming conventions](http://kodi.wiki/view/Naming_video_files/TV_shows) in
order to succesfully index TV shows in its library. 

Now imagine that you move out to go to college and scramble together
a small machine to use as a HTPC. The cpu is fine, the gpu is fine, but
the storage capacity is not nearly as much as you’ve grown accustomed to.
You can’t mount the server at home as remote storage because your Austrian
roommate will forever nag you about the traffic messing up his Dota game
and the connection isn’t fast enough to stream anyway. So what do you do?

I figured that the most logical solution was to just rsync all the files
in the directory to the local machine until the volume was full, starting
with file that was modified last. Seems easy right? How about something
with 

{% highlight bash %}
find . -type f -printf '%T@ %p\n'
{% endhighlight %}

Just pipe that into sort and xargs to rsync the files individually to the
HTPC until the volume is full. Unfortunately its not that simple. Synology
machines are equipped with Busybox binaries instead of proper GNU
utilities. This means that there is no -printf flag on the find or the
xargs command, the stat command is severely limited and the default shell
is [Almquist shell](http://en.wikipedia.org/wiki/Almquist_shell). This
limits the options and convenience quite a bit. Some fancy glob shelling
trick won’t be possible too. A savage option would be to pipe something
from ls, but even that was off the table due to the not functioning ‘-e
List full date and time’  flag of the BusyBox implementation. A compromise
could be made where the ‘-mtime’ find flag could be used to simply get all
the files not older than a certain amount of time but I didn’t like the
idea of having the hdd just sitting there idely until new files came by.

The next step could be to look into options like utilizing the [Debian
chroot](http://www.mdevries.org/synology_debian_chroot.html) or to find
some other way to get better binaries on the system. None of that seems
like the ‘clean’ way to do it but luckily there is another route to go.

My solution consists of two scripts. [One Python script that runs on the
Synology
server](https://github.com/vdloo/dotfiles/blob/master/code/scripts/system/latestfilter.py)
to list all the files in the source directory and to create symlinks to
the latest files in a temporary directory until a certain amount of bytes
of data is reached. [The second
script](https://github.com/vdloo/dotfiles/blob/master/code/scripts/system/latestsync.sh)
is a small shell script that runs on the remote machine.

This way every time the latestsync.sh script is run on the small server,
a directory is made on the large server, the dir is filled up with
symlinks to files sorted by age in a copy of the original directory
structure from the specified path until this new temporary directory has
reached the allocated amount of bytes specified in the script. Then rsync
on the smaller server will start pulling everything from that directory
down to the local volume. 

To set it up drop the latestfilter.py script somewhere on the remote
server like for example

{% highlight bash %}
wget https://raw.githubusercontent.com/vdloo/dotfiles/master/code/scripts/system/latestfilter.py -P /volume1/RAID5/other/code/scripts/system
{% endhighlight %}

On the local server download the latestsync.sh script to some directory

{% highlight bash %}
wget https://raw.githubusercontent.com/vdloo/dotfiles/master/code/scripts/system/latestsync.sh -P ~/.dotfiles-public/code/scripts/system/
{% endhighlight %}

And run the script once in a while using a cron job like this based on
your configuration

{% highlight bash %}
*       7,12,17,22      *       *       *       SHELL=/bin/bash PATH=/bin:/sbin:/usr/bin:/usr/sbin bash /home/vdloo/.dotfiles-public/code/scripts/system/latestsync.sh -h example.com -u vdloo -p 22 -a 1500000000000 -l /mnt/lvmdisk/series > /dev/null 2>&1
{% endhighlight %}
