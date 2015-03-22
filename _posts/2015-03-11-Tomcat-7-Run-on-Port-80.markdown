---
layout: post
title:  "Run Tomcat7 on Port 80"
date:   2015-03-1 19:46:32
categories:
---

You may need to run Tomcat on port 80 for many reasons. It is possible to accomplish this in three steps. <br>
First step is move 8080 to port 80.Find your server.xml file for Tomcat. It is probably under /etc/tomcat/server.xml but still 
you can find it by using: <br>
{% highlight bash %}
$ locate server.xml
/etc/tomcat7/server.xml
{% endhighlight %}
Open server.xml file with an avaliable text editor (I will use nano).
{% highlight bash %}
$ sudo nano /etc/tomcat7/server.xml
{% endhighlight %}
Find the part where port is defined as 8080 and change it to 80.
{% highlight xml %}
 <Connector port="80" protocol="HTTP/1.1"
               connectionTimeout="20000"
               URIEncoding="UTF-8"
               redirectPort="8443" />
{% endhighlight %}
Close the file by saving it. If you run tomcat now it will complain about some authentication errors. It happens because we are trying to run 
Tomcat under 1023 ports. So the next step is enable authbind for tomcat. If you run Tomcat on port numbers that are all higher than 1023, then you
 do not need authbind.  It is used for binding Tomcat to lower port numbers. Open /etc/default/tomcat with your text editor.
{% highlight bash %}
$ sudo nano /etc/default/tomcat7
{% endhighlight %}
Switch authbind=no to authbind=yes which is located at the end of file.
{% highlight xml %}
# NOTE: authbind works only with IPv4.  Do not enable it when using IPv6.
# (yes/no, default: no)
AUTHBIND=yes
{% endhighlight %}
The last step is disabling IPv6. You can see the warning just before authbind line saying that "NOTE: authbind works only with IPv4.  Do not enable it when using IPv6.".
Open /etc/sysctl.conf with your text editor.
{% highlight bash %}
$ sudo nano /etc/sysctl.conf
{% endhighlight %}
And add these three lines to end of the file.
{% highlight xml %}
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
{% endhighlight %}
Save the file and close it. To make sure IPv6 is disabled run 
{% highlight bash %}
sudo sysctl -p
{% endhighlight %}
and you will see 
{% highlight bash %}
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
{% endhighlight %}
After that, if you run:
{% highlight bash %}
$ cat /proc/sys/net/ipv6/conf/all/disable_ipv6
{% endhighlight %}
It will report: 1 <br>
Now it is ready to run on port 80. Start your tomcat with 
{% highlight bash %}
$ sudo service tomcat7 start
{% endhighlight %}
If you look at your catalina.out logs it will be written in there as 
{% highlight bash %}
Mar 21, 2015 10:02:44 PM org.apache.coyote.AbstractProtocol init
INFO: Initializing ProtocolHandler ["http-bio-80"]
{% endhighlight %}

Now you can access your services directly without any port like <br>
example.com/Services/


 
