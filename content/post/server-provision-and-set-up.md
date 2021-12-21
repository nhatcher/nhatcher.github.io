---
title: "Server Provision and Set Up"
date: 2021-12-19T23:46:46+01:00
Description: ""
Tags: []
Categories: []
DisableComments: false
Draft: false
---

# How to provision a set up a server for a web application

WARNING: Work in progress

Once you have a web application you will need to deploy it to a server. You will learn here how to do that.
This is for you if you have a small web application that connects to a database of sorts and has to serve dynamic content.
Here you will learn in it's simplest form how to take that application and deploy it in the internet. Give it a URL, make it secure and even get an email associated to your service.

If you want a landing page, a blog or just your home page there are simpler and almost free ways of doing it.

There are are only two things you will need to pay for:

* Hosting (<10$/monthly)
* Domain Name (~10$/year)


## Providers

The first thing you need is a computer were your application is running. That comes with the name of a VPS (Virtual Personal Server).
You could host it yourself if you can have a computer permanently connected to the internet.
I will show you how to do that some other time, here today we will pay for the service.

There are many good companies that offer a good service for 5 to 10 € a month.
Some I have tried:

- [Scaleway](https://www.scaleway.com/en/)
- [Digital Ocean](https://www.digitalocean.com/)
- [Linode](https://www.linode.com/)
- [Hetzner](https://www.hetzner.com/)
- [Hostinger](https://www.hostinger.com/vps-hosting)

Remember to set the ssh keys and get an Unbuntu instance.

## Secure server

Note on terminology. We will assume that the user in our account is `John Smith`, their user name is `jsmith`,
their website is `http://www.example.com` and their ip `93.184.216.34`.

Once the server is provisioned we can ssh into it:

```bash
jsmith@my_computer:~/$ ssh root@93.184.216.34
```

First thing we do is install fail2ban:
```bash
root@93.184.216.34:~# apt install fail2ban
```


### User administration.

The first thing we are going to do is to create users and grant them sudo access.

Add user and create bash, home folder, etc:
```bash
root@93.184.216.34:~# adduser jsmith
```
You will need to add a temporary password for the user. If you are the user, then you don't need to change the password afterwards.
Otherwise communicate the user the password in a secure manner.

Add the user to the sudoers list
```
root@93.184.216.34:~# usermod -aG sudo jsmith
```

Create authorized_keys so they can ssh without passwords
```bash
root@93.184.216.34:~# su jsmith
jsmith@93.184.216.34:/root$ cd /home/jsmith/
jsmith@93.184.216.34:~$ mkdir .ssh
jsmith@93.184.216.34:~$ chmod 700 .ssh/
jsmith@93.184.216.34:~$ cd .ssh
jsmith@93.184.216.34:~$ touch authorized_keys
jsmith@93.184.216.34:~$ chmod 700 authorized_keys
```
And copy the users public (id_rsa.pub) in the file. Using vim or nano.

Then form the remote computer you should be able to ssh without a password:

```bash
jsmith@my_computer:~/$ ssh root@93.184.216.34
...
jsmith@93.184.216.34:~$
```

They should change their password
```bash
jsmith@93.184.216.34:~$ passwrd
```

Repeat for any superuser in the system.

### Firewall

WARNING: Be very careful in this section, if you make a mistake you can lock yourself out permanently from the computer.

In this section we are going to setup a _firewall_. That's basically a simple way for blocking traffic.
This is a bit of a complicated issue. The still standard way of doing it in Linux is with a  program called `iptables`. That's a bit too complex for a beginner and there are a couple of programs that will simplify your life while using iptables in the background. Two I have used are _firehol_ and _ufw_. We will use here the later (although I actually use the former in most situations).

Install the _uncomplicated firewall_ and make sure it is disabled.

```bash
root@93.184.216.34:~# apt install ufw
root@93.184.216.34:~# ufw disable
```

We will:
* reset all rules
* deny all incoming traffic
* allow all outgoing traffic
* allow all traffic for ssh (port 22), http (port 80) and https (port 443)

We run the commands:
```bash
root@93.184.216.34:~# ufw default deny incoming
root@93.184.216.34:~# ufw default allow outgoing
root@93.184.216.34:~# ufw allow ssh
root@93.184.216.34:~# ufw allow http
root@93.184.216.34:~# ufw allow https
```

Finally enable the firewall:
```bash
root@93.184.216.34:~# ufw enable
```

NOTES: Even iptables is outdated it's successor since 2014, _nftables_ is still not widely used. They both (iptables and nftables) run on the top of a kernel space program called _netfilter_. How all this works is something for a later (more theoretical and interesting post).

## Logs

You can find logs in...

## Webserver and the Application Servers

If you have build your web application you probably run it on your local machine in some port with some command.
It might be python, node js, go,... any language. Some times you might need to run more than one of those to have the application running.
I call those the _application server_. Depending a bit on your development environment they might *not* be production ready.
For instance, many people like their webpages to reload _automatically_ whenever there is a file change. That's definitely not fine for production purposes (both the application server and the javascript build). Simple python servers like bottle or django are not ready to be deployed on production.
What is the right application sever for your particular case it is difficult for me to say, it depends on your programming language and your stack.
In general if you are working with python you might want to use [Gunicorn](https://gunicorn.org/) or [uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/). If you are working with node the famous _express_ might be production ready. A simple server written in the _go_ programming language might need nothing else.

But even when you have selected that, the application server might benefit a lot for a professional _webserver_ sitting in front.
The webserver can do everything that is language agnostic and that does not need to be repeated for every language and framework. Things like compress files, do the SSL, route general traffic, cache files. The webserver can also help serving static files like javascript, html, css or images. In general the webserver will sit in front, being the first (after the firewall, of course) in getting all the traffic directed towards your computer and it will be up to it what to do with the request, route it to some other service or act by serving a static asset. The webserver takes care of so many other things you probably don't want to think about like slow clients, lost of request coming at the same time, dealing with wrong requests. And even beyond, the webserver can be used as a load balancer, manipulate response headers, doing access control and security checks, fetching storage backends to decide where to route a request,... List is probably endless.

There many fine and battle tested webservers. Three I have used extensively are: [Apache](https://httpd.apache.org/), [nginx](https://www.nginx.com/) and [Caddy](https://caddyserver.com/).

My personal favourite is Caddy, but we will use nginx this time around. I do not use Apache anymore, but it is a fine choice. 

Before anything you should make sure there is not a webserver already installed in your system. A simple test would be to visit your ip address `https://93.184.216.34/`.
If there is a webserver already installed you should remove it first. It is most likely the Apache webserver.

First let's install nginx and serve some static content.

```bash
root@93.184.216.34:~# apt install nginx
root@93.184.216.34:~# service start nginx
```

Now if you visit your ip address you should find the default nginx greeting page.

```
location / {
        # First attempt to serve request as file, then
        # as directory, then fall back to displaying a 404.
        try_files $uri $uri/ =404;
        root /var/www/html;
}

location /api/ {
        proxy_set_header Host $host;
        # proxy_set_header X-Real-IP $remote_addr;
        proxy_pass http://127.0.0.1:8080/;
}

```

## Services

We will first create an underprivileged user to run the web application.
We created some normal users before, with `home` and added them to the sudoers list. We are going to go the opposite way here.


```bash
# vim /lib/systemd/system/my_app.service
```

Example service

```
[Unit]
Description=My description of the app
Documentation=https://example.com/documentation
After=network.target

[Service]
Type=simple
User=node
WorkingDirectory=/var/www/server
ExecStart=/opt/node/bin/node /var/www/server/index.js
Restart=always

[Install]
WantedBy=multi-user.target
```

To start it

```bash
# service my app start
```


## Domain names

Namecheap

## Certificates

Let's encrypt

## Getting email

Zoho

