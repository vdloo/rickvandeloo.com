---
title: Suspend and continue processes (ctrl+z) in a shell script
layout: post
---

If you have spend some time in a command line environment you have probably run into the problem of wanting to run multiple programs at once. In a unix-like shell you can do this in many different ways. You could open up a second TTY (ctrl+alt+f2), use a terminal multiplexer like screen or tmux, or you can launch the process in the background from the beginning by appending an ampersand (command &). Often it is after you have already launched the program that you realize you need to run another command from the same terminal. To do this, you can first suspend it with the key combination (ctrl+z) and then push it to the background with bg or disown it with the disown command if your shell supports that.
<br>

Control-z is very interesting because it suspends the process you are running, effectively pausing it. It works by sending the TSTP signal to the process. This can be very convenient if you just need to clear up some processing power or have another reason to give your process a break before continuing (like for example pausing tar if your volume is about to have no space left). The Control-z functionality can be simulated by running kill -TSTP $PID with $PID as the process ID of your process. Just like you might resume the process with fg in the command line, you can run kill -CONT $PID to un-pause the process. So if you were to implement this in a shell script, you could do something like 
<br>

{% highlight bash %}
$ command & COMMANDPID=$!
{% endhighlight %}   

and then do
<br>

{% highlight bash %}
$ kill -TSTP $COMMANDPID
{% endhighlight %}   

to pause your process after initial execution ($! catches the PID of the last launched process in the shell, if you get "-bash: !: event not found" add a whitespace after the !). The problem with this approach is that this functionality is mostly useful when you are running parts of your script parallel and already in the background or in subshells, and if that is the case then doing kill -TSTP might not be good enough. 
<br>

You can run a script or program with lots of subprocesses or subshells and neatly pause it with Control-z. If you'd try the same thing with the kill -TSTP command in a complex shell script you might run into some difficulties. The difference here is that kill -TSTP only sends the signal to the highest PID in the hierarchy and the signal won't reach all the levels. This is often not a problem because most scripts don't go that deep, if you want that level of abstraction for system stuff you are probably better of with some scripting language like python or perl. But hey, in the spirit of why not: to make sure you are actually suspending the entire process chain you can do something like this.
<br>

{% highlight bash %}
$ for childprocess in $(ps -o pid --ppid $PID | tail -n +2); 
	do kill -TSTP $childprocess; 
  done;
{% endhighlight %}   

replace TSTP with CONT to continue the chain. Watch out though, the --ppid flag (select by parent process ID) is not supported on FreeBSD so if that's your platform you might have to figure out something else. 
