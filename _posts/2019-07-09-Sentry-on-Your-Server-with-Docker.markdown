---
layout: post
title:  "Setting up Sentry on Your Server with Docker"
date:   2019-07-09 11:41:05
categories:
---
If you want to use Sentry on your server with your custom domain then it is quite easy. Main setup will have: 

* nginx for reverse proxy: requests coming to sentry.my_domain.com will be redirected to sentry application runnning in docker. 
* docker for deploying sentry
* ubuntu 18.04 or 16.04 server (mine was 18:04 from [https://www.hetzner.com/cloud](https://www.hetzner.com/cloud) but DigitalOcean or Vultr all works). 
* a domain or possibly subdomain which is already redirecting to our server ip. can check details from [https://lmgtfy.com/?q=redirect+subdomain+to+ip+address](https://lmgtfy.com/?q=redirect+subdomain+to+ip+address)
* letsencrypt for ssl


On Ubuntu 18:04 server, start with installing docker (can skip if you've installed):

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

Letsencyrpt for SSL (When certbot asks for nginx settings it doesn't matter if you choose 1 or 2. We will update the nginx setting files afterwards):

{% highlight bash %}
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt install python-certbot-nginx
$ sudo certbot --nginx -d sentry.your_domain.com
{% endhighlight %}

Before moving forward first disable docker from updating iptables since we want to use ufw as a simple firewall. What does this mean? Simply when you use docker and expose a port to the server (can be used for any reason) docker doesn't care about ufw. Let's say we will use these simple firewall settings and block everything else.
{% highlight bash %}
$ ufw allow https
$ ufw allow ssh
$ ufw allow http
{% endhighlight %}

What you expect now that if docker is running a web app at port 8080 you should not be allowed to access to it by ip:8080. Well unfortunately docker will override rules including the firewall and expose this port to global. If you want to read details about this check [https://www.mkubaczyk.com/2017/09/05/force-docker-not-bypass-ufw-rules-ubuntu-16-04/](https://www.mkubaczyk.com/2017/09/05/force-docker-not-bypass-ufw-rules-ubuntu-16-04/) 
To prevent this:
{% highlight bash %}
$ echo "{
\"iptables\": false
}" > /etc/docker/daemon.json

$ sed -i -e 's/DEFAULT_FORWARD_POLICY="DROP"/DEFAULT_FORWARD_POLICY="ACCEPT"/g' /etc/default/ufw
{% endhighlight %}

Restart docker and ufw (allow ssh-http-https as shown above). 
{% highlight bash %}
$ sudo service docker restart
$ sudo ufw reload
{% endhighlight %}
Setup sentry using docker in these order. Migrating take some time.
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

   If you were not asked for the superuser while upgrading database then create a new one:
{% highlight bash %}
   $ docker run -it --rm -e SENTRY_SECRET_KEY='generated_key_from_above' --link sentry-redis:redis --link sentry-postgres:postgres sentry createuser
{% endhighlight %}

Go to `$ cd /etc/nginx/sites-enabled/` and rm the default setting file if its there with `$ rm default`. Add the nginx file to that files with any name. `$ nano sentry`
{% highlight bash %}
 server {
    listen   80;
    server_name sentry.your_domain.com;
    set_real_ip_from 127.0.0.1;
    set_real_ip_from 10.0.0.0/8;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;
    root /var/www/html;
    location ~ /.well-known {
        allow all;
    }


    location / {
      if ($request_method = GET) {
        rewrite  ^ https://$host$request_uri? permanent;
      }
      return 405;
    }
  }

  server {
    listen   443 ssl;
    server_name sentry.your_domain.com;

    proxy_set_header   Host                 $http_host;
    proxy_set_header   X-Forwarded-Proto    $scheme;
    proxy_set_header   X-Forwarded-For      $remote_addr;
    proxy_redirect     off;

    # SSL configuration -- change these certs to match yours
    ssl_certificate /etc/letsencrypt/live/sentry.your_domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/sentry.your_domain.com/privkey.pem;

    # NOTE: These settings may not be the most-current recommended
    # defaults
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:128m;
    ssl_session_timeout 10m;


    # keepalive + raven.js is a disaster
    keepalive_timeout 0;

    # use very aggressive timeouts
    proxy_read_timeout 10s;
    proxy_send_timeout 10s;
    send_timeout 10s;
    resolver_timeout 10s;
    client_body_timeout 10s;

    # buffer larger messages
    client_max_body_size 5m;
    client_body_buffer_size 100k;

    location / {
      proxy_pass        http://localhost:9000;

      add_header Strict-Transport-Security "max-age=31536000";
    }
  }

{% endhighlight %}

Go to your domain on sentry.your_domain.com and can login there. 