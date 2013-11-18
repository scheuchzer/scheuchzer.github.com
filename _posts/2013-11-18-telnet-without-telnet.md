---
layout: post
title: "Telnet without Telnet"
description: ""
category: Linux
tags: [Bash, Telnet, Connectivity, Firewall]
---
{% include JB/setup %}

Recently I had to verify the connectivity of a new server. So, I logged in over SSH
and simply typed 

{% highlight bash %}
telnet my.server.com 5432
{% endhighlight %}
  
to test the firewall rules. But to my suprise, telnet was not installed on that machine.
 
So, what now?

Ok, we could check if stuff like **curl** or **wget** is installed but this won't help 
in every case. If you simply want to know if a certain port is open you can use the 
command: 

{% highlight bash %}
exec 3> /dev/tcp/my.server.com/5432;[ $? == "0" ] && echo ok || echo fail
{% endhighlight %}

Not as neat as **telnet** but does the job :-)
