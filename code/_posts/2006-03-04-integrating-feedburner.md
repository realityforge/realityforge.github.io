--- 
layout: post
title: Integrating FeedBurner
---
I have been using Google Analytics to track my site traffic but up until now I have been not tracking who is accessing this site from my Atom and RSS Feeds. So I have started to publish my feeds through FeedBurner.

However I want the transition to be as smooth as possible and not interfer with users who had already subscribed to my feed. Initially I decided to redirect all of my feeds to FeedBurner by adding the following to my `.htaccess` file. This redirected all requests except those with a user agent of FeedBurner to the appropriate FeedBurner web site.

{% highlight apache %}

1.  Redirect typo feeds to FeedBurner
    RewriteCond %{HTTP\_USER\_AGENT} !^FeedBurner$
    RewriteRule ^xml/(atom|rss|rss20)/feed.xml$
    http://feeds.feedburner.com/RealityForge \[R,L\]
    {% endhighlight %}

However this did not feel right as readers were being redirected to another web site. So I decided to reverse proxy the content from FeedBurner on my site. What this means is that anyone who accesses these urls is actually accessing content from the FeedBurner web servers but it appears as if it comes from my web site. My web server actually forwards the request to the feedburner website, gets their response and then returns it to the client.

{% highlight apache %}

1.  Redirect typo feeds to FeedBurner
    RewriteCond %{HTTP\_USER\_AGENT} !^FeedBurner$
    RewriteRule ^xml/(atom|rss|rss20)/feed.xml$
    http://feeds.feedburner.com/RealityForge \[P,L\]
    {% endhighlight %}

Of course things aren’t always as easy as this. Unfortunately my web server is behind a firewall and another proxy server. So I needed to add the following config to my main apache config file.

{% highlight apache %}
ProxyRequests On
ProxyVia On

ProxyRemote http://feeds.feedburner.com/RealityForge
http://proxy.cs.latrobe.edu.au:8080
{% endhighlight %}

This forwards any requests for http://feeds.feedburner.com/RealityForge to my local proxy server to handle.

And it all seems to work now!
