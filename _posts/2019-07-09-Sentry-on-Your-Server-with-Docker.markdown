---
layout: post
title:  "Setting up Sentry on Your Server with Docker"
date:   2019-07-09 11:41:05
categories:
---
If you want use Sentry on your servers with your custom domain then it is quite easy. Main setup will have: 

* nginx for reverse proxy: requests coming to sentry.my_domain.com will be redirected to sentry application runnning in docker. 
* docker for deploying sentry
* ubuntu 18.04 or 16.04 server (mine was from  [https://www.hetzner.com/cloud](https://www.hetzner.com/cloud) but DigitalOcean or Vultr all works). 

* a domain or possibly subdomain already redirecting to our server ip. can check details from   [https://lmgtfy.com/?q=redirect+subdomain+to+ip+address](https://lmgtfy.com/?q=redirect+subdomain+to+ip+address)
* letsencrypt for ssl


On Ubuntu 18:04 server start with installing docker:

{% highlight bash %}
$ sudo apt update
$ sudo apt install apt-transport-https ca-certificates curl software-properties-common
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
$ sudo apt update
$ sudo apt install docker-ce
{% endhighlight %}

Install nginx

{% highlight bash %}
$ sudo apt install nginx
{% endhighlight %}

Letsencyrpt for SSL (When certbot asks for nginx settings it doesn't matter if you choose 1 or 2. We will update the nginx settings files afterwards):

{% highlight bash %}
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt install python-certbot-nginx
$ sudo certbot --nginx -d sentry.your_domain.com
{% endhighlight %}

Before moving forward first disable docker from updating IPTOOLS since we want to use ufw as simple firewall. What does this mean? Basically when you use docker and expose a port to server (can be used for any reason) docker doesn't care about ufw. Let's say we will use these simple firewall settings and block everything else.
{% highlight bash %}
$ ufw allow https
$ ufw allow ssh
$ ufw allow http
{% endhighlight %}

What you expect now that if docker is running a web app at :8080 you should not be allowed to access to it by ip:8080. Well unfortunately docker will override rules including the firewall and expose this port to global. If you want to read further about this check link https://www.mkubaczyk.com/2017/09/05/force-docker-not-bypass-ufw-rules-ubuntu-16-04/
To prevent this:
{% highlight bash %}
$ echo "{
\"iptables\": false
}" > /etc/docker/daemon.json

$ sed -i -e 's/DEFAULT_FORWARD_POLICY="DROP"/DEFAULT_FORWARD_POLICY="ACCEPT"/g' /etc/default/ufw
{% endhighlight %}

And restart docker and ufw (allow ssh-http-https as shown above). 
{% highlight bash %}
$ sudo service docker restart
$ sudo ufw reload
{% endhighlight %}
Setup sentry using docker in these order 
{% highlight bash %}

   $ docker run -d --name sentry-redis redis
   $ docker run -d --name sentry-postgres -e POSTGRES_PASSWORD=your_postgres_password -e POSTGRES_USER=sentry postgres
   $ docker run --rm sentry config generate-secret-key
{% endhighlight %}

keep the generated secret_key to use in the next steps.
{% highlight bash %}

   $ docker run -it --rm -e SENTRY_SECRET_KEY='generated_key_from_above' --link sentry-postgres:postgres --link sentry-redis:redis sentry upgrade

   $ docker run -d -p 9000:9000 --name custom-sentry -e SENTRY_SECRET_KEY='generated_key_from_above' --link sentry-redis:redis --link sentry-postgres:postgres sentry

   $ docker run -d -p 9000:9000 --name custom-sentry -e SENTRY_SECRET_KEY='generated_key_from_above' -e SENTRY_SINGLE_ORGANIZATION=false -e SENTRY_USE_SSL=0 --link sentry-redis:redis --link sentry-postgres:postgres sentry   

   $ docker run -d --name sentry-cron -e SENTRY_SECRET_KEY='generated_key_from_above' --link sentry-postgres:postgres --link sentry-redis:redis sentry run cron
   $ docker run -d --name sentry-worker-1 -e SENTRY_SECRET_KEY='generated_key_from_above' --link sentry-postgres:postgres --link sentry-redis:redis sentry run worker
   {% endhighlight %}

   If you were not asked for the superuser while upgrading databasae then:
{% highlight bash %}
   $ docker run -it --rm -e SENTRY_SECRET_KEY='generated_key_from_above' --link sentry-redis:redis --link sentry-postgres:postgres sentry createuser
{% endhighlight %}

Go to your sentry.your_domain.com and you can login there. 