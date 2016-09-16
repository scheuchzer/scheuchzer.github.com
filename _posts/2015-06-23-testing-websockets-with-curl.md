---
layout: post
title: "Testing WebSockets with CURL"
description: ""
category: "linux"
tags: [Bash, JavaScript, Curl]
---
{% include JB/setup %}

[Socket.io]: http://socket.io/

Just played around with [Socket.io] to access a backend service over WebSockets. As you might guess, it didn't worked right from the beginning.

So, I wondered if there's a way to test the backend with the good old `curl` command. And yes! There is!

{% highlight bash linenos %}
$ curl -i -N \ 
	-H "Connection: Upgrade" \
	-H "Upgrade: websocket" \
	-H "Host: localhost:8080" \
	-H "Origin:http://localhost:8080" \
	http://localhost:8080/chat
{% endhighlight %}

After starting, `curl` will wait and dump all messages that the server sends.
