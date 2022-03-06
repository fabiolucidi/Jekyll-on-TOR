#  Introduction
I started this project/guide a couple of days after the beginning of the Ukrainian war, when I realized Russia government was adopting new measures to limit the _free access to free information_ to the Russians citizens. If you can't access independent sites to verify the news, your government is afraid of what you can do with those information and this is exactly why being informed is so fundamental for a human being: to be able to choose what to do and how to live.
If reading and being informed is important, having the right to inform without being persecuted is even more important: and this is what this guide is about, being able to post news in a censorship resistant way using TOR and without using any third party services.

If you're here you've probably heard of _TOR_ (**T**he **O**nion **R** outing), one of the most helpful tools we have to fight censorship. If you don't know that TOR is or if you want to learn more please visit the [TOR Project page](https://www.torproject.org/).

In this guide I'll show how to run your site on the TOR network using a free software called [Jekyll](https://www.jekyllrb.com).

# System configuration
## System Specs
The hardware requirements are very low, any computer can do the job, but since we're talking about having a website you may want to have it online 24/7. This could raise an energy issue (power, heating, noise, etc.). The best hardware solution - simple and cheap - is a [Raspberry Pi](https://www.raspberrypi.com/), a cheap single-board computer. It's not easy to find it at the moment due to the chip/supply chain crisis, but if you have it it'd be perfect. Other wise you can run on any other pc hen you turn it on: it's better to provide free information for just 1h per day rather than no information at all.

All the commands I've used can be used on other Debian based Linux distribution, like Ubuntu and are easilly adaptable to Mac OsX (_unchecked_).

## SSH Login for Raspberry Pi
In case you're using a Raspberry Pi I strongly suggest to activate the SSH login. I'd suggest this in any case, but specifically in this case since our computer will be exposed on the internet (aka people will connect to _our_ device).

A perfect guide to do this configuration is [this one from the RaspiBolt project](https://raspibolt.org/guide/raspberry-pi/security.html).

Once you're done with the SSH Login configuration login to your raspberry with your admin account (usually pi):
ssh pi@raspberrypi

If you're going to use your regular pc you can ignore the above mentioned procedure but please consider using strong measures to protect your laptop because your posts will be stored there. Keep in mind that if the UFW configuration for your personal pc would be different from the one described later here: consider to customize it as you need to be able to connect to other services (internet, etc.).

##  Packages installation
### Security
For everything related to the security of your _server_ you can refer to [this page](https://raspibolt.org/guide/raspberry-pi/security.html#enabling-the-uncomplicated-firewall) that will guide step-by-step in the installation and configuratino process of four important packages we will need: **uncomplicated Firewall**, **fail2ban**, **NGINX**.
### TOR
To install TOR simply run
```
$ sudo apt install tor
```
### Jekyll
```
$ sudo apt-get update
$ sudo apt-get install software-properties-common -y
$ sudo apt-get install ruby-full -y
```

## Create a dedicated user
We'll create a dedicated user for the jekyll installation and we'll call it jekyll. We'll configure it with the `--disabled-password` option
```
$ sudo adduser --gecos "" --disabled-password jekyll
```
And we will add jekyll user to tor group
```
$ sudo adduser jekyll debian-tor
```

A dedicated user is not fundamental, but if you run more than one service on the same machine it would be useful. Moreover, it helps keeping things clean and removes the risk to accidentally delete your files or folder since they're separated from your regular account.

If you want to use your regular account you should add it to tor group
```
$ sudo adduser yourusername debian-tor
```
## Reverse Proxy
NGINX will help us managing all the connections in a particular way called _reverse proxy_. If you want to read more about it you can start [here](https://www.nginx.com/resources/glossary/reverse-proxy-server/).

We'll create our jekyll-reverse-proxy.conf file:
```
$ sudo nano /etc/nginx/streams-enabled/jekyll-reverse-proxy.conf
```
```
upstream jekyll {
  server 127.0.0.1:4001;
}

server {
  listen 4001 ssl;
  proxy_pass jekyll;
}
```
Once we've saved the file we need check the configuration and reload.
```
$ sudo nginx -t
$ sudo systemctl reload nginx
```
## Configure Firewall
Let's configure the firewall to allow incoming requests and only those we want:
```
$ sudo ufw allow 4001/tcp comment 'allow Jekyll SSL'
```
And check if the instruction is ok with:
```
$ sudo ufw status
```

## TOR
Configuring TOR could be tricky, but not this time :) We literally just need to add a three lines section and get our Onion address.
So first things firts, let's add the following lines in the “location-hidden services” section in the torrc file.
```
$ sudo nano /etc/tor/torrc
```
```
### This section is just for location-hidden services ###
HiddenServiceDir /var/lib/tor/hidden_service_jekyll/
HiddenServiceVersion 3
HiddenServicePort 80 127.0.0.1:4001
```
Save and exit.
Then we need to reload Tor config and get our onion address:
```
$ sudo systemctl reload tor
$ sudo cat /var/lib/tor/hidden_service_jekyll/hostname
> abcdefg..............xyz.onion
```
Now our system configuration is complete and we've our onion address that we'll share to those we want to read our site. We just need to build one.


## Run Jekyll as a non super user
Now it's time to work on our site, so we need to login with our dedicated user:
```
$ sudo su - jekyll
```
To run Jekyll as a non super user we need to add these lines to the `.bashrc` file (it's in the home folder).
`$ nano .bashrc`
```
---
# Ruby exports
export GEM_HOME=$HOME/gems
export PATH=$HOME/gems/bin:$PATH
---
```
Then run
`$. .bashrc`
to reload it. And then:
```
$ gem install jekyll bundler
```
To install (almost) everything we need.

This tells `gem` to place its gems within the user’s home folder, not in a system-wide location, and adds the local `jekyll` command to the user’s PATH ahead of any system-wide paths.

## Build your site
If you're starting from scratch you can just run:

`$ jekyll new myswebsite`
and then
`$ cd myswebsite`
to access the folder.

If you want to import something already published on GitHub Pages (or another Jekyll site) to create a _TOR Version_ of that website, you need to create a folder and access it
```
$ mkdir mywebsite
$ cd myswebsite
```
and then import the content from your repository with
```
git clone https://github.com/yourname/yourproject.git ~/mywebsite
```
In any case, after that, you need to install your site dependencies (Gems, etc.) running
```
bundle install
```
and then build and execute the site with:
```
bundle exec jekyll serve --port 4001 --detach
```
**Now your website is live and accessible via TOR** from the onion address we've fetched before (in the example: abcdefg..............xyz.onion).

### Jekyll as a service
A very useful hint is running jekyll as a service instead of a command. [This script](https://gist.github.com/yuan3y/51f6534c9daaa2f64baa64e1a3c361aa) from
@yuan3y will do the job.

To do that run
```
$ sudo nano /etc/systemd/system/jekyll.service
```
Cut and paste the following code specifying your jekyll folders and the username, save and exit.

```
# Author: @yuan3y
# Date: 2017-09-29
# Description: to make `jekyll serve` a system service and start on boot
#
# Usage: place this file at `/etc/systemd/system/jekyll.service`
# then run
#  sudo systemctl start jekyll.service
#  sudo systemctl enable jekyll.service

[Unit]
Description=Jekyll service
After=syslog.target network.target

[Service]
User=***[my user name]***
Type=simple
WorkingDirectory=***[my jekyll source folder]***
ExecStart=/usr/local/bin/jekyll serve --watch --source "***[my jekyll source folder]***"
ExecStop=/usr/bin/pkill -f jekyll
Restart=always
TimeoutStartSec=60
RestartSec=60
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=jekyll

[Install]
WantedBy=multi-user.target network-online.target
```
Then start and enable the service with
```
$ sudo systemctl start jekyll.service
$ sudo systemctl enable jekyll.service
```
