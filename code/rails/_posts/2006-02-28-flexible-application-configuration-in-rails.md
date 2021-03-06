--- 
layout: post
title: Flexible Application Configuration in Rails
---
Most of my rails application have required some form of application specific configuration. When the configuration varies depending on where I deploy it I prefer to externalize the configuration in a YAML file similar to the way database configuration is handled.

I remember reading a blog somewhere that suggested the use of [OpenStruct](http://www.ruby-doc.org/stdlib/libdoc/ostruct/rdoc/classes/OpenStruct.html) and have since started using the following code placed at the bottom of my `config/environment.rb` file.

{% highlight ruby %}
require ‘ostruct’
require ‘yaml’

ConfigFile = “\#{RAILS\_ROOT}/config/config.yml”
if File.exist?(ConfigFile)
::ApplicationConfig = OpenStruct.new(YAML.load\_file(ConfigFile))
end
{% endhighlight %}

Consider the scenario where `config/config.yml` contains;

{% highlight yaml %}
proxy\_config:
host: proxy.cs.latrobe.edu.au
port: 8080
{% endhighlight %}

I could then access the the configuration data using

{% highlight ruby %}
&gt;&gt; ApplicationConfig.proxy\_config\[‘host’\]
=&gt; “proxy.cs.latrobe.edu.au”
&gt;&gt; ApplicationConfig.proxy\_config\[‘port’\]
=&gt; 8080
{% endhighlight %}

To make your code more robust in the scenario where either the proxy configuration is missing or the whole configuration file is missing I generally add in extra guards. Below is an example of the code I use to instantiate a Net:HTTP object. If the configuration file is present and it contains a proxy\_config section then the code will attempt to instantiate a Net::HTTP::Proxy object. Otherwise the code will instantiate a plain old Net::HTTP that will not attempt to go via a proxy server.

{% highlight ruby %}
def http
if Module.constants.include?(“ApplicationConfig”) &&
ApplicationConfig.respond\_to?(:proxy\_config)
host = ApplicationConfig.proxy\_config\[‘host’\]
port = ApplicationConfig.proxy\_config\[‘port’\]
Net::HTTP::Proxy(host, port)
else
Net::HTTP
end
end
{% endhighlight %}
