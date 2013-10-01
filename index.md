---
layout: page
title: Java-Adventures.com
tagline: Supporting tagline
---
{% include JB/setup %}

<p>
&nbsp; A blog about Java and other stuff.
</p>
<hr/>

{% for post in site.posts %}
  <div class="hero-unit">
    <p>
    	<h2>
    		<a href="{{ post.url }}">{{ post.title }}</a>
    	</h2>
    	{{ post.date | date: "%A %d %B %Y" }}
    	<p>{{ post.content | strip_html | truncatewords: 55 }}</p> 
    	<p><a href="{{ post.url }}">Read more ...</a></p> 
    </p>
    <hr/>
  </div>
{% endfor %}

