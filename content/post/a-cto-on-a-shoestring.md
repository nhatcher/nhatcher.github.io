---
title: "A CTO playbook on a shoestring"
date: 2023-10-15T12:15:35+02:00
Description: ""
Tags: []
Categories: []
DisableComments: false
---

All the code in this blog post is in [github](https://github.com/nhatcher/users-template) and you should have a look at that first and come here only for further assistance.

This post was **last updated on Sun, Oct 16, 2023**

In this blog post you are going to learn how to setup and deploy a simple web application with modern best programming practices and all the bells and whistles you need to grow the app from zero to a few hundred users.
There might be some things you want to do differently from code in here. You might want to use a different platform to read your logs, you might want to use NGINX instead of caddy, you might want to run your django code with Daphne instead of gunicorn, you might decide GitHub is not good for you, or use a different email provider or host provider or get the domain name from a different name registrar. You will still be able to use this with minimal modifications. Changing Django and python for a different framework or programming language will require more changes but the core of this post will still be useful. Using [Lavarel](https://laravel.com/), [Ruby on Rails](https://rubyonrails.org/) or plain [go](https://go.dev/) should pose no big challenges to the reader.

This should not only be regarded as a tutorial on how to setup and install everything but as a guide on how to build a modern web application. Using the code provided and following this guide might get you up and running in a matter of hours instead of days or even weeks but you should be prepared to do some sleuthing.

The guide is intended for a small team, maybe just one single person. If your team is larger than 5 people this guide might still work but might fall short in some respects. I am assuming you have almost no cash to dedicate to this project, we are going to do this on a shoestring. With all this in place if you make the project grow from a solo developer and no users to a few hundred users and 5 developer you will very easily upgrade the tools and hardware. I am assuming also you have some background in programming and you are comfortable in a Linux terminal. If you are not you really need a more technical partner that will help you with that. You will also not learn Django or python here although you should be ok if you know just some python and are willing to dedicate some time go go through the Django documentation.

Although I am guiding you through the cheapest possible option, as soon as you start getting customers you should consider investing in infrastructure. Maybe you will need a separate database server o a more performant server or paying for some of the services you are using.

This is not the only way to setup a server. There are tons of other ways of doing this. Some might want to use [Heroku](https://www.heroku.com/) or [Fly.io](https://fly.io/) or a completely managed [Cloudron](https://www.cloudron.io/). There are of course the big enterprise solutions like [Google Cloud](https://cloud.google.com/), [AWS](https://aws.amazon.com/) or [Microsoft Azure](https://azure.microsoft.com/)

Lastly we will not be setting the frontend here. All we are going to do here is completely frontend agnostic. You could do the whole frontend in vanilla HTML, CSS and JavaScript or use any modern framework like React and TypeScript.

After having setup toy webapps following similar patterns, the idea of this project came when I joined forces with Lu Bertolucci to work on his project [Feirou](https://www.feirou.org/). As I had some available time in my hands I thought this was a good moment to do this right. 

Many of the things that I will recommend here I learn from my coworkers during the past 14 years! I also have them to thank.

A brief note on nomenclature. I will assume that your username is 'jsmith', that your local's name computer is 'local', your remote name computer is 'remote', the name of the domain you bought is 'example.com' and that the ip in your VPS is '93.184.216.34'.
If you are following in Linux or Mac you will have no issues. If you are following in a Windows machine and are working in the WSL it should be pretty straightforward. Otherwise you might need to tweak some commands for your own environment and OS. When you see a command like:

```
root@remote# apt install failban
```
means that you are running this command as root in the remote machine. And the remote folder is not important. Note that you know that it is a root user because the prompt ends with '#' and not with '$'. With this provisions I hope it is clear where are you running what command.

Finally, I will check links and versions. I will try not to link to third party blogs but prefer tools documentation. That said, links might be broken. I will always try to provide information about the link so you can search the internet easily.

## Technology stack and options

This is a quick survey of the technologies we are using, why I chose them and the alternatives.

1. __Web server: Caddy__ server. Runner up NGINX.
    
    We picked Caddy vs NGINX, because Caddy is even simpler to configure than NGINX. Caddy is just one binary, you can run it in your local machine and it has https build in. I am a big fan. NGINX is also great, you can't go wrong with any of them.

    Apache is still a valid option.

2. __Framework: Python + Django.__ No runner up, or just too many of them.

    Most of the technology, advices and best practices are fairly independent from  Python and Django so you could easily adapt this to _Nodejs + Express_, for instance. If you are comfortable with Python but you don't want to use Django, you might use Flask.

    Some other programming languages like Java (and the JVM languages) or C# have it's own ecosystem and will have implications on what other tools you use.
3. __Application server: Gunicorn.__ Runner up Daphne.

    Gunicorn follows the WSGI server protocol while Daphne follows ASGI. This basically means that Gunicorn is a blocking synchronous server and Daphne is asynchronous.

    We went for Gunicorn for simplicity. Upgrading to a ASGI compliant server should be an easy job. That's tomorrows problem.

    Other options include uWSGI, Uvicorn (ASGI), Hypercorn (ASGI). See the [Django documentation](https://docs.djangoproject.com/en/4.2/howto/deployment/)
4. __Database: PostgreSQL.__ Runner up MySQL or SQLite
    
    If you have a reason why not to use PostgreSQL go ahead and do that. Note that we are still using SQLite during development.
5. __Host provider: DigitalOcean.__ Runner up Scaleway or Oracle Cloud (free!)
    
    There are a lots of great options here. We went with DigitalOcean because it is cheap and an excellent service. Second to none.
6. __Domain registrar: Namecheap.__ Runner up GoDaddy

    Again there are tons of options here. Take into account that not all name registrars offer the same feature. Exercise caution when going for a less known one. Also keep an eye for the renewal costs.
7. __Email provider: Zoho.__ Runner up Google (paid service).

    Other options <https://www.migadu.com/> for 20€ a year or [prontonmail](https://proton.me/) for 50€/year for 10 email addresses. Fastmail has a similar price to Google, [tutanota](https://tutanota.com/pricing) has also a very generous offer and might even be for free if you are doing [open source](https://tutanota.com/discount).
    A completely different way would be to self host. I don't recommend that, although you will have infinite amount of email addresses and will be in control of everything you might find that many of your emails are not being delivered correctly. As of today email is a sort of oligopoly and you might end up in an unlucky situation in which all your outgoing email is being blocked. There are people with notable success here, so it is definitely possible. If you are going down that route consider using [Cloudron](https://www.cloudron.io/) with a DigitalOcean droplet.
8. __Logs: Sentry.__ Runner up Loggly
    I don't have a ton of experience here. Other two options are [new relic](https://newrelic.com/) and [Loggly](https://www.loggly.com/).
    I chose Sentry for simplicity, it would be very easy to switch to another platform. Your logging platform should be mostly unintrusive.
9. __Wireframes: [Draw.io](https://www.drawio.com/).__ Runner up Figma
    Although this blog post is not about the frontend, it helps a lot having the wireframes in place. Draw.io is a simple, free, well done tool useful in many contexts. One of the many things I learned from Stephen Grider.
    Figma can also be used for wireframing. And also lots of other possibilities.
10. __GitHub__. No runner up
    There are alternatives out there like [GitLab](https://about.gitlab.com/), but I haven't used them
11. __Ubuntu Linux__. No runner up
    We will be deploying in an Ubuntu VPS. Any debian based system should work in a very similar way. The whole thing will work in any Linux system but you might have to change a few commands.


## Getting a domain name (~10€ a year)

You can buy a domain name from places like [namecheap](https://www.namecheap.com/), but there are many other vendors. Just make sure that they only sell you the domain name and nothing else. It should cost you around 10€ a year. But you can find really cheap domain names for a year you can get for your testing. Beware of not renewing them, those cost go up and add!

## Getting an email account

There are several paid vendors like Google. But we will use [Zoho](https://www.zoho.com/mail/zohomail-pricing.html) that for now is free. Scroll down to the "forever free plan". You can use one provider now and move to a different one in the future. Because you own the domain name you will not need to change email addresses. The only problem you might have to deal with is exporting from Zoho and importing them again in the new platform if you ever want to do that.

Zoho has a nice web service and apps for Android and iOS to read and send your email from the phone.

To setup your email, you will need to follow the steps in one of these guides, [namecheap](https://www.namecheap.com/support/knowledgebase/article.aspx/9758/2208/how-to-set-up-zoho-email-for-my-domain/) or [zoho](https://www.zoho.com/mail/help/adminconsole/namecheap.html).

With this done you should be able to send a receive emails with your new fancy email address. You can up to 5 email accounts, for instance `hello@example.com` and `support@example.com` but you can configure zoho to get all emails on `hello@example.com`. You can also now send emails programmatically.

Now you should be able to send an email programmatically:
```python
import smtplib
from email.mime.text import MIMEText
from email.header import Header
from email.utils import formataddr

# Configure in zoho
password = "my secret password"
domain = "example.com"
sender_user = f"support@{domain}"


def send_email(email):
    # Define to/from
    subject = "First email ever!"
    sender_title = "Example App Support"

    # Create message
    msg = MIMEText(
        "Hello world!",
        "plain",
        "utf-8",
    )
    msg["Subject"] = Header(subject, "utf-8")
    msg["From"] = formataddr((str(Header(sender_title, "utf-8")), sender_user))
    msg["To"] = email

    # Create server object with SSL option
    server = smtplib.SMTP_SSL("smtp.zoho.com", 465)

    # Perform operations via server
    server.login(sender_user, password)
    server.sendmail(sender_user, [email], msg.as_string())
    server.quit()

if __name__ == "__main__":
    # who do you want to send the email to:
    to_address = "jsmith@test.com"
    send_email(to_address)

```

## Server provision and setup (~5€ a month)

This is by far the most complicated bit. But you only have to do it once.

Before even getting a VPS you need a pair of ssh keys. You probably have already done it for [GitHub](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent).

Your cloud provider will probably let you [upload your public key](https://docs.digitalocean.com/products/droplets/how-to/add-ssh-keys/to-team/) so that every new VPS will have the ssh key installed.

I am using an instance in [DigitalOcean](https://www.digitalocean.com/) basic droplets that at the time of writing are advertised as 4$/month but because of billing issues turns out to be around 4.5€ a month. This is enough for our purposes.
Other options are [Scaleway](https://www.scaleway.com/en/), [Linode](https://www.linode.com/), [Hetzner](https://www.hetzner.com/), and many others. Most notably [Oracle Cloud](https://www.oracle.com/cloud/free/) offers a forever free tier!

Once you have your own VPS the only important thing is that you have a set of two scripts:

* `install.sh` that you need to run once when you provision the VPS.
* `deploy.sh` that you need to run every time you want to update your code.

This section is just an explanation of what [`install.sh`](https://github.com/nhatcher/users-template/blob/main/deployment_scripts/install.sh) does.

1. Secure the server

Log into the remote server. That is normally done by:

```
jsmith@local:~/ $ ssh root@93.184.216.34
```
Where `93.184.216.34` is just the example IP.


Install `fail2ban`:
```
root@remote# apt install fail2ban
```

Make sure you can only ssh with a key and not with a password. So `ssh root@93.184.216.34` only works form your computer with the private ssh key.

Note that playing with firewall is a little dangerous, not only you can expose your machine to the rest of the internet if you do something wrong, you can also you lock yourself out if you set the wrong rules. Some vendors give you a direct console access to your VPS and that will mitigate the issue. But if you don't have direct console access tread carefully. So read all the documentation first and apply the rules that are convenient only after understanding what you are doing.

A firewall is a piece if software, or hardware, that acts as security barrier monitoring and controlling incoming and outgoing network traffic based on a set of predefined rules or policies. The Linux kernel has a build in [Netfilter](https://en.wikipedia.org/wiki/Netfilter) framework that we can control in userspace with a command-line utility, 'iptables', that allows you to configure and manage firewall rules and network packet filtering. In layperson terms this means that at the Operative System level we have a way to check all data coming in or going out of your computer.

Raw 'iptables' commands are hard to understand, even for experts and there are other programs designed to generate this rules for you. A good example is the 'fail2ban' command above. In Ubuntu system the program 'UFW', (the Uncomplicated FireWall) is widely used. The following will deny all incoming traffic and allow all outgoing, then allow all ssh, http and https. Hopefully it is self explanatory.
```
root@remote# apt install ufw
root@remote# ufw disable
root@remote# ufw default deny incoming
root@remote# ufw default allow outgoing
root@remote# ufw allow ssh
root@remote# ufw allow http
root@remote# ufw allow https
root@remote# ufw enable
```

And if that worked 100% of the cases I wouldn't have had to tell you about 'iptables' in the first place. If you are administrating a VPS chances are you will have to see the face of the tiger sooner rather than later.

If you can't use UFW a set of rules that works well for our purposes is:
```bash
#!/bin/bash

# Flush existing rules
iptables -F

# Allow incoming SSH (port 22) traffic
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow established connections
iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# Allow incoming HTTP (port 80) traffic
iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Allow incoming HTTPS (port 443) traffic
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow loopback (localhost) traffic
iptables -A INPUT -i lo -j ACCEPT

# Allow ICMP (ping) traffic
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Set the default policies to DROP (block all other incoming traffic)
iptables -P INPUT DROP
iptables -P FORWARD DROP

# Allow all outgoing traffic
iptables -P OUTPUT ACCEPT
```
You can run the commands one by one or run in in a file `firewall-setting.sh`.
If you want to make a backup of your current rules before you do your foolishness:
```
# iptables-save > ~/iptables-rules
```
And to restore them:
```
# iptables-restore iptables-rules
```

You need to know that applying iptables rules are not persisted after boot. So if you do get locked out just reboot your VPS. There are several ways to persist the rules. Creating a system script like we have done for caddy and gunicorn is not a bad way:
```
[Unit] 
Description=runs iptables restore on boot
ConditionFileIsExecutable=/opt/util/restore-iptables.sh
After=network.target

[Service]
Type=forking
ExecStart=/opt/util/restore-iptables.sh
start TimeoutSec=0
RemainAfterExit=yes
GuessMainPID=no

[Install]
WantedBy=multi-user.target
```

And then the `restore-iptables.sh` script would be:
```
#!/bin/sh
/usr/bin/flock /run/.iptables-restore /opt/util/iptables-restore /etc/iptables/rules.v4
```

Enough with the firewall. Create a user and add it to the sudoers list. Make sure you can ssh with that user and remove ssh root access to the machine.

```
root@remote# adduser jsmith
```

Where 'jsmith' is your username. Add it to the sudoers list:

```
root@remote# usermod -aG sudo jsmith
```

As `jsmith` copy the contents of the root's `authorized_keys` file (usually at `/root/.ssh/authorized_keys`) to the username's `authorized_keys` file (`/home/username/.ssh/authorized_keys`). In a different terminal test that you can ssh as the new username:

```
jsmith@local:~/ $ ssh jsmith@93.184.216.34
```

Once in the remote computer check that you can become a superuser by typing `sudo su` and entering your password. Now keep your password safe and delete the root's `authorized_keys` file.

Congratulations, you have secured your server. You should not be able to ssh as root anymore.

Now, there are still services running on your computer you might not be aware of. One of them is the OSI layer 3 echo ICMP server built in the operative system. For your laptop at home try:

```
jsmith@local:~/ $ ping -c 5 example.com
PING example.com (93.184.216.34) 56(84) bytes of data.
64 bytes from example.com (93.184.216.34): icmp_seq=1 ttl=52 time=26.0 ms
64 bytes from example.com (93.184.216.34): icmp_seq=2 ttl=52 time=30.1 ms
64 bytes from example.com (93.184.216.34): icmp_seq=3 ttl=52 time=24.4 ms
64 bytes from example.com (93.184.216.34): icmp_seq=4 ttl=52 time=25.7 ms
64 bytes from example.com (93.184.216.34): icmp_seq=5 ttl=52 time=27.8 ms

--- example.com ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 24.359/26.784/30.122/1.988 ms
```

This is helpful to test if your computer is up and the network is listening. Note that we have not allowed the service in the firewall, yet it is there running. This is telling you that a round trip time for  ping request to your machine is around 26 milliseconds.

Your cloud provider might also feature some firewall settings. I don't think that setting a firewall both at the os level and at the cloud provider level would break anything but it's probably not a great idea since it might lead to difficult to debug issues. But you should go ahead and read about the firewall in your cloud provider and play with it a little bit. If you are using DigitalOcean in the control panel select Networking->Firewalls->Create Firewall. Once created you can _assign_ the firewall to the droplet in the firewall page. As usual DigitalOcean shines on good documentation. You could set some rules and see how they affect your droplet. Generally speaking you should accept the ssh, https and icmp inbound rules and allow all outbound communication.

If you are of FreeBSD this section will the only one that is a bit different for you. Luckily for you FreeBSD has not one but [three firewalls](https://docs.freebsd.org/en/books/handbook/firewalls/)!

2. Point your domain name to your remote computer

We want your ip to point to your remote computer, this will buy us two things:

* We will be able to do `ssh jsmith@example.com` instead of using the ip
* Your friends will be able to visit `www.example.com` and land to your webpage!

We do this by configuring the DNS records in your DNS host provider. This is not difficult, but I bet you will have some stories to tell if you spend enough time configuring those.
Normally your domain registrar will provide you with a full featured GUI to config those DNS records in the _zone file_.

The only thing you have to do is to add a couple of `A Record`'s in your advanced DNS settings. If you are using Namecheap will look something like:
![namecheap config](/images/shoestring/namecheap_config.png "Configuring DNS records")

Note that we are adding three `A record`s:
* *`@` Record* directs the root domain ('example.com') to your IP
* *`www` Record* directs the subdomain 'www.example.com' to your IP
* *`app` Record* directs the 'app.example.com' to your IP 

We will do that to have our webpage sitting in <https://www.example.com> and our web application sitting in <https://app.example.com>.

You don't need to do this. You can have everything in the root domain <https://example.com> like <https://twitter.com> or <https://stackoverflow.com>. Having a different subdomain for the application can help with a number of issues:
* You can have other services in other subdomains independent of the application
* The webpage subdomain and the application subdomain do not share any cookies

On the other hand if you don't have other services or you have other domains for them and you don't need a flashy landing page you can go and use the root domain.

At the end of the day wether you want to have the application sitting in your root domain or a subdomain is your decision.

You don't need to point all the `A Records` to the same computer. For example your webpage could be host in a different place like [GitHub](https://pages.github.com/). See the documentation bellow about the Caddyfile.


3. Set up a simple TLS secured webserver.

We will now install a webserver in your VPS capable of serving static webpages and redirecting traffic. Apache and NGINX are fine options but will be using [caddy](https://caddyserver.com).

First thing you should do is download the latest binary for your computer architecture. You can follow the instructions [in the caddy documentation](https://caddyserver.com/docs/install) but we will install the latest binary here:

```
root@remote# wget https://github.com/caddyserver/caddy/releases/download/v2.7.4/caddy_2.7.4_linux_amd64.tar.gz
root@remote# tar -xf caddy_2.7.4_linux_amd64.tar.gz
root@remote# cp caddy /opt/caddy/
root@remote# chmod +x /opt/caddy/caddy
```

Remember that if you do this you will ned to maintain caddy version's yourself and be specially aware of security updates.

We will follow [caddy](https://caddyserver.com/docs/running).

Add an underprivileged user:

```
root@remote# groupadd --system caddy
root@remote# useradd --system --gid caddy --create-home --home-dir /var/lib/caddy --shell /usr/sbin/nologin --comment "Caddy web server" caddy 
```

Create two directories:
```
root@remote# mkdir /var/www/api/
root@remote# mkdir /var/www/website/
```

Now get an `index.html` file in each of them:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Website/App</title>
</head>
<body>
The app is sitting in the website/app endpoint
</body>
</html>
```

Make sure that everything is owned and readable by caddy:

```
root@remote# chmown -R caddy:caddy /var/www/
```

Create a Caddyfile:
```
example.com {
    redir https://www.example.com{uri}
}

www.example.com {
	root * /var/www/website/

	file_server
}

app.example.com {
	root * /var/www/app/

	file_server
}
```
This is redirecting all traffic from 'example.com' to 'www.example.com'. Anything coming from 'www.example.com' is being served from the directory `/var/www/website/` and 'app.example.com' is being served from `/var/www/app/`. As simple as that. In this case both subdomains are served from the same VPS, but that doesn't need to be the case. <https://www.example.com> could be served form a different machine, maybe done in wordpress or in modern days in webflow or anything else. You could use a static site generator like [Hugo](https://gohugo.io/) or [Zola](https://www.getzola.org/) or build it yourself if you are brave and host it in GitHub or [Neocities](https://neocities.org/). If you are doing that remember to update the DNS record in your name registrar.

Finally run caddy:
```
root@remote# /opt/caddy/caddy run
```

If you did everything alright and I didn't forget any instructions, by visiting <https://example.com> you should be redirected to <https://www.example.com>. If you visit <https://www.example.com> or <https://api.example.com> you should see your two different html files.

Big congratulations! Note: everything should be __out of the box https__ and not http. This is one of the big advantages of using Caddy. You can, of course, use other webservers like NGINX and configurations will not be much more difficult.

This is all good and dandy. But we can't keep running caddy from the terminal like we are doing now, we need to run it as a service.

To do that create a `/etc/systemd/system/caddy.service` with the following content:
```
[Unit]
Description=Caddy
Documentation=https://caddyserver.com/docs/
After=network.target network-online.target
Requires=network-online.target

[Service]
Type=notify
User=caddy
Group=caddy
ExecStart=/opt/caddy/caddy run --environ --config /etc/caddy/Caddyfile
ExecReload=/opt/caddy/caddy reload --config /etc/caddy/Caddyfile --force
TimeoutStopSec=5s
LimitNOFILE=1048576
LimitNPROC=512
PrivateTmp=true
ProtectSystem=full
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

Copy the Caddyfile to `/etc/caddy/Caddyfile`.

Then run to reload the system daemons and the caddy daemon
```
root@remote# systemctl daemon-reload
root@remote# systemctl start caddy.service
```

The controversial systemd is used across all Linux operating systems and is very powerful but somewhat complex. In you are on FreeBSD you can use the [BSD-style init](https://docs.freebsd.org/en/articles/rc-scripting/)

Now you are running Caddy as a service and can shut down your laptop and go for pizza or beer because you had your first deployment. Your system is up and running!

One final note. In this post we are only interested in the 'app.example.com' part. But for us it will be a bit more complicated because we will not be just serving plain static html files. We will need Caddy to work as a proxy server. We will fix that issue in the coming sections.

4. Setup the Postgres database

In your production machine install Postgres and dependencies:

```
# apt install libpq-dev postgresql postgresql-contrib
# apt install build-essential python3-dev
```

Log into an interactive Postgres session by typing:

```
# su postgres
$ psql
```

Create a database in the Postgres prompt. Words in angle brackets ("<>" symbols) are placeholders that you should replace with actual values. For example, "\<database-user\>" could become "jsmith":

```
postgres=# CREATE DATABASE <database-name>;
postgres=# CREATE USER <database-user> WITH PASSWORD '<database-password>';
postgres=# ALTER ROLE <database-user> SET client_encoding TO 'utf8';
postgres=# ALTER ROLE <database-user> SET default_transaction_isolation TO 'read committed';
postgres=# ALTER ROLE <database-user> SET timezone TO 'UTC';
postgres=# GRANT ALL PRIVILEGES ON DATABASE <database-name> TO <database-user>;
postgres=# GRANT ALL ON SCHEMA public TO <database.user>;
```
Here we just follow the recommendations in the [Django documentation](https://docs.djangoproject.com/en/4.2/ref/databases/#postgresql-notes). The last command is needed in PostgreSQL 15. 

You can, of course use a different database like MariaDB, MySQL, Oracle, CockroachDB, Firebird, Microsoft SQL Server or even MongoDB. We will be using SQLite in local development. Please have a good reason if you don't want to use PostgreSQL.

5. Create an underprivileged django user:

This is the user that will run the 
```
root@remote# groupadd --system django
root@remote# useradd --system --gid django --create-home --home-dir /var/lib/django --shell /usr/sbin/nologin --comment "Django app runner" django
```

If your remote branch is private you will need some _deployment keys_ for the user _django_.
```
root@remote# su - django -s /bin/bash -c 'ssh-keygen -t ed25519 -C "Deployment key" -N "" -f ~/.ssh/id_ed25519'
```

You should add that key to your [GitHub remote repository](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys#deploy-keys). Make sure it does not have writing permissions.



## Local development and architecture

Now that we've successfully set up the remote machine and it's all set for our application, let's focus on how we'll be developing the application locally.

What I'm about to share isn't exactly the standard approach, but it's a method I personally prefer for local development. In a modern web application, you typically have two key components: the backend and the frontend. Normally, both of these components have their own development servers. These development servers aren't meant for production use, but they offer some useful features like Hot Module Reloading (HMR). So, during development, most folks end up running two local servers on their machines, each listening on different ports (one for the frontend and one for the backend).

There are various ways to manage this, but I find a straightforward solution is to use Caddy as a proxy server. It's worth noting that Caddy is a lightweight tool – just a single binary. You won't need to install any hefty dependencies or deal with strange configurations on your laptop. To kickstart your workday, all you need to do is open a terminal and run the following commands.

```
jsmith@local:~/Project/example$ caddy run
```

The Caddyfile could be something like:
```
:2080

# django API
reverse_proxy /api/* localhost:8000

# admin API
reverse_proxy /admin/* localhost:8000
reverse_proxy /static/admin/* localhost:8000

# everything else is the frontend 
reverse_proxy :5173
```

That means the frontend is running on port 5173 and the Django backend will be running on port 8000 but our whole application will be running on port 2080. Now in a different terminal head over to the frontend folder and run:
```
jsmith@local:~/Project/example/frontend_test$ python -m http.server 5173
```

Today, this will serve as our frontend server. In a future installment of this series, we'll replace it with a more sophisticated, React-aware, HMR-enabled, and cache-friendly development frontend server.

For now, these two terminals won't cause much interference in our workflow.

You might be wondering why we have those two entries in the Caddyfile related to the Django API. We'll dive into their purpose later, and we'll also explore how to handle them in a production environment.

In production, we'll have a setup quite similar to this, where we'll differentiate between requests for static files and API requests. Here's how it works: a remote machine, say a browser, will send requests to our server located at <https://app.example.com>, for example. These requests fall into two categories:

1. Static content: This includes JavaScript files, HTML, CSS, images – essentially public resources.
2. API calls: These requests require server-side processing.

Some websites are solely of the first type – they simply serve the content you request. Personal blogs and websites mostly fall into this category. You can even host them for free on platforms like GitHub or [Neocities](https://neocities.org/) among others. They don't involve login, database queries, or e-commerce functionality. These are known as static websites and are relatively straightforward to create.

The second type of requests necessitates a server or program to process data and return relevant information. For instance, a search engine needs to provide a list of URLs containing specific information. Many of these API calls are public and open for anyone to use, while some might be private, accessible only to users who have logged into the system.


## The app design and wireframes

Although today's aim is not the frontend we will build a frontend to have a fully functioning application. When designing an app or a website about the first things you need to do is the _wireframes_. Those are the low fidelity designs of what the application will do. Wireframes are not necessarily done by designers, they will probably came afterwards creating pixel perfect designs of your ideas.

There are many wireframing tools out there, but I will not mention any of them as I will leave that for our frontend discussions. For our purposes I will create the diagrams we need with <https://www.drawio.com/>.

Our web application will be as simple as possible. Users can:

* Log in and see, update and delete their personal data
* Log out
* Create accounts
* Recover password via email

Just the bare minimum that a world changing app needs to have. The frontend will be all done in HTML, modern JavaScript and CSS. No frills.

This are the wireframes:
![example app wireframes](/images/shoestring/ExampleApp.svg "Wireframe for a simple App")

When the user goes to our website <https://app.example.com>, three things can happen:
* If they are not logged in, you will see screen (1)
* If they are logged in as a normal user they will be greeted in screen (2)
* If they are admins they will see the list of users in the platform (2B)

I think the rest are pretty self explanatory. Notice that at this point we do not specify every single detail:
* What happens if you if you type a incorrect password?
* The real implementation might be quite different after a designer sees some of the wireframes and decides to change the UX here and there
* The exact wording will change
* ...

Now you can go over the [frontend_test](https://github.com/nhatcher/users-template/tree/main/frontend_test) folder and see the code there. We will not discuss it here.

## The database design
We finally get to the backend code and design. The first thing we need to think about is the database structure.

We obviously need a table with users that has to store username, an encrypted password, name, last name, email, etc.

The users table might also include pictures, a telephone number and what not.
Things start getting more complicated when you realize you need to store session data. How do we know a user is already logged in? Also sites should be protected from _cross-ste request forgery_ or CSRF. And probably a couple of other issues we haven't thought just yet. That's when _django_ jumps in. Now, we are not going to design the authentication part of the database ourselves, we could, and we will probably do that on another occasion, but if we are using django, that is done for us. Of course Django's database schema might not be exactly what we want and we will need to adapt it for our own purposes.

We will extend Django's 'auth_users' with our own 'user_profile' table that will have a one to one link with Django users. This way we can add some extra information to every user without having to change the internal system. Also the framework doesn't have a way to create accounts or recover passwords. We will need to design that.

When someone creates an account it will be disabled until they click on the link that was send by email.
The way we will do this is to have a new table (model in the parlance of Django) called 'pending_users' that has a one to one link to a 'user' that is initially disabled. It will also contain the 'email_token'.

Another table 'recover_password' will contain the emails and email tokens send to those users that asked to recover their password.

And that's pretty much it for this project. In production we will be using a PostgreSQL database and we will use Sqlite3 in development.

## Setting up the django project

To begin with let's create a new folder for our application:

```
jsmith@local:~/Projects/$ mkdir example-app
jsmith@local:~/Projects/$ cd example-app
```

Now create a virtual environment for the project:

```
jsmith@local:~/Projects/example-app$ python -m venv venv
```

Where the second venv could be any name you like

Activate the virtual environment:

```
jsmith@local:~/Projects/example-app$ source venv/bin/activate
```

Install django:
```
(venv) $ pip install django
```
You will need to think a bit about the directory structure you want. In our case we want two directories in the root directory of out project ‘frontend’ or client and ‘server’. We will go ahead and create those two directories and move to the sever directory:
```
(venv) $ mkdir server
(venv) $ cd server
```

Now we create a django project in that directory:
```
(venv) $ django-admin startproject settings .
```

Note two things. We call the project ‘settings’. That is because we want to have a settings directory for the django admin. Also note the period and the end. This is to create the project in our current directory. We call it settings so that django places all config stuff in there.

There is a bit of configuration we still need to do. First we are going to use different setting for production and local development:

* We don't want to send real emails
* We don't want to send logs to our log backend. We will probably be happy seeing the emails  and the logs in our terminal.
* We don't want a PostgreSQL database while development. SQlite3 will do just fine.
* We want HMR while development and full stack traces
* We don't need access to secret keys

See the [source code](https://github.com/nhatcher/users-template/tree/main/server/settings/settings) to understand the specifics.



## The users or accounts app

A Django project revolves around the concept of apps. A Django application is a self-contained unit that encapsulates specific functionality within a Django project. Django apps are designed to be reusable and can be plugged into different projects, which makes them a fundamental building block of Django web development.

Almost every project is going to need an accounts or users app. The bulk of the code we will write today is in this app. This is our first stop:
```
(venv) $ python manage.py startapp users
```
In this app we need to define:
1. *Models*: Models define the structure and behavior of the database tables for your app. Each model corresponds to a table in the database and includes fields (representing table columns) and methods for interacting with the data. In our example, "user_profile," "pending_users," and "recover_password" are all appropriate names for models. Models are a critical part of your app, as they determine how data is stored and manipulated.
2. *URLs (URL Routing)*: The URLs module in your app defines the URL patterns or routes for your app's endpoints. These patterns map specific URLs to views within your app.
3. *Views*: Views are responsible for handling HTTP requests and returning HTTP responses. They contain the logic for processing data, rendering templates, or returning data in formats like JSON. Views determine what happens when a specific endpoint is accessed. It's essential to provide details about how views interact with the URL routing and how they respond to different HTTP methods (e.g., GET, POST).
4. *Tests*: everything within the app should be thoroughly tested. This is a crucial aspect of Django development
5. *Admin*: The admin module is where you can define custom configurations for your app's models within the Django admin interface. You can specify how your models should be displayed and edited by administrators. This is particularly useful for managing app-specific data through the admin panel.

Once all the code is in place we are ready to test the app locally! Just need to create the migrations and run them:

```
(venv) $ python manage.py makemigrations
(venv) $ python manage.py migrate
```

Once that is done if your caddy proxy server is running and your python frontend server is running you just need to:
```
(venv) $ python manage.py runserver
```
And head over to <https://localhost:2080/> to see your application. You should also see the Django admin page at <https://localhost:2080/admin/>

Note the database file `db.sqlite3` will be sitting along side `manage.py` and you can check it. You can open that file with the `sqlite3` cli, use a sqlite browser like [SqliteStudio](https://sqlitestudio.pl/) or simply use a VS Code [Sqlite Viewer](https://marketplace.visualstudio.com/items?itemName=qwtel.sqlite-viewer). In either case to be able to log in the Django admin panel you will need to be a superuser. Using the CLI:

```
$ sqlite3 db.sqlite3 
SQLite version 3.37.2 2022-01-06 13:25:41
Enter ".help" for usage hints.
sqlite> UPDATE auth_user SET is_superuser=TRUE WHERE username="jsmith";
```

Or you can create a superuser with:
```
(venv) $ python manage.py createsuperuser
```

As stated above tests are a crucial component of a Django application and, in fact, are vital for any computer program. They are essential for ensuring that the software works correctly and reliably, regardless of the specific technology or framework being used. Ti run the tests here:
```
(venv) $ python manage.py test
```

Remember that the `manage.py` uses the development settings by default.

Typically there might be as many lines of code for tests as there are for production code. And you will need to think them thoroughly. Focus on testing the logic first, not on lines of code covered. It's very easy, although sometimes cumbersome, to test every line of the code but to miss important logical branches of the code. It is very easy to skip tests on a first proof of concept, thinking you might rewrite this properly in the future. In my experience that is not normally how things go down. You start writing a bit of code for a proof of concept and that code ends ups growing out of hand with a bad foundation. A good part of the reason for this blog post and code is to help you get started in a project with a solid foundation.

You should spend some time understanding how the tests work. In many situations you are going to find yourself thinking a piece of code is difficult to test. That might be so, refactor, make the thing testable. At times you will need to mock a part of the application, we don't want to send real emails in tests or we don't really have a browser. You will need to think how to do that in a clean way. Specially at the beginning of the development you shouldn't focus on test coverage, trying to cover every line of code and it is ok to leave some parts out rather than install complicated frameworks that will help you testing weird corner cases. Sometimes tests that way end up being a heavy luggage. As long as the testing framework is in place and you have an easy system to add new tests and you are covering the main logical branches of your code you are ready to take off.

Enough about tests for now, we will meet them again in a later section dedicated just for them.


## The type checker, the formatter and the linter

Python is showing it's age and doesn't have many of the features that are now taken for granted in more modern languages. In particular it doesn't have types, meaning it's very easy to call a function with incorrect types, for example. Because of this there is a growth in tools that intend to patch the issues in different ways. Picking a particular set of tools and configuring them correctly might be overwhelming.

What I describe here is just one selection, as long as you are checking types, adhering to a style guide and having a code formatter.

We will use [pyright](https://microsoft.github.io/pyright/#/) by Microsoft for the type checker. There are many other options, [mypy](https://www.mypy-lang.org/) is the reference type checker but mymy seems to be having many issues with Django.

We will use [black](https://github.com/psf/black) for our code formatter and [flake8](https://flake8.pycqa.org/en/latest/) to enforce a style guide. We will also use [isort](https://pycqa.github.io/isort/) to order our imports consistently.

You will need to run this tools every time you commit code to the repository and make sure that code is conformant.

```
(venv) $ pip install pyright
(venv) $ pip install black
(venv) $ pip install flake8
(evnv) $ pip install isort
```

I leave the configuration options and settings for you, but you can find ours in [.flake8](https://github.com/nhatcher/users-template/blob/main/.flake8) and [pyproject.toml](https://github.com/nhatcher/users-template/blob/main/pyproject.toml). Note that you need to run isort in the black compatibility mode.


## Tests, test runner, test coverage and GitHub actions

We will use [pytest](https://docs.pytest.org/) and [pytest-cov](https://pypi.org/project/pytest-cov/). To install this:
```
(venv) $ pip install pytest
(venv) $ pip install pytest-django
(venv) $ pip install pytest-cov
```

This is a nicer way to run the tests than running `python manage.py test` for a number of reasons, richer test discovery, more concise syntax, you can do parametrized testing, parallel test execution, custom test makers and use plugins like the coverage tool we will be using.

Running:
```
(venv) $ pytest --cov-report term-missing --cov server
```

Will run all your tests, run a test coverage and tell you which lines are not yet covered.

This is the important bit, all test need to pass before merging more work into the main branch. This includes not only the Django tests but also the type checker, formatter and linters. I have put all of them in the [run_test.sh](https://github.com/nhatcher/users-template/blob/main/run_tests.sh) executable.

## Logs, alerts, notifications and stats

Logs are messages emitted by the application or the operative system that can help us trace back errors or issues that happened through an event or series of events.
It can be informative, signal a warning or be worrisome errors.

Alerts are messages that we receive in our phones. We probably need to act on them. They signal a house on fire event.
Notifications are also messages that we receive in our movil devices that are just for our information, for example new accounts created.

Some logs may be updated to alerts or notifications depending on their importance.

Stats are numbers of certain occurrences that we feel are important. Like number of accounts created or the number of times people logged in into the system.

## Deployment if the repository is private

We will need to use [Deploy keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys#deploy-keys)

## Refinements:

1. Cron jobs:
* Clean database

2. unattended updates. Caddy updates

3. Vaults and secrets. Terraform vault.

4. Staging server(s)
You might want to have a staging server in <https://staging.example.com>. If you want to be safer you can do it using random subdomains: <https://sleigh_year.g.example.com>. You can copy the data from production. Droplets are cheap for a few hours or days.

5. Creating mock data

When testing the app it is very useful to have the app filled mock data. For that we have a script that fills the database and returns a file with users and passwords.


6. What a second, what is deployed again?

Having a visible deployed commit id will save you from trouble.

## Extra A: A VPN with Wireguard

## Extra B: Deployments on tagging

## Extra C: Code defines infrastructure.

Using Terraform and Puppet



