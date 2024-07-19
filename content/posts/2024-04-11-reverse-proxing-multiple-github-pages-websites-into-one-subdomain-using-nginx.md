---
title: Reverse-proxing multiple GitHub Pages websites into one subdomain using NGINX
description: Using NGINX to reverse-proxy multiple static websites from GitHub Pages to a single subdomain
date: 2024-04-11
author: Federico Mahfoud
---

## Context

GitHub provides good hosting for static websites. I'm using it for:
- A personal landing page: [fedemahf.github.io/federico.mahfoud.ar](https://fedemahf.github.io/federico.mahfoud.ar)
- A resume: [fedemahf.github.io/resume](https://fedemahf.github.io/resume)
- And a blog: [fedemahf.github.io/blog](https://fedemahf.github.io/blog)

But that's not enough for me. I want all my static websites to live under my domain, [federico.mahfoud.ar](https://federico.mahfoud.ar).

I decided it was about time to dive into NGINX. I searched for a "getting started" kind of video and found [NGINX Tutorial for Beginners](https://www.youtube.com/watch?v=9t9Mp0BGnyI) by *freeCodeCamp*. This video opened my eyes and made me understand that the `proxy_pass` directive was what I needed.

## Prerequisites

- A domain - in my case it's `federico.mahfoud.ar`
- A static website - can be GitHub Pages
- A VPS - I used [OVH's Starter VPS](https://www.ovhcloud.com/en/vps/) (4.20 USD/month)
- Basic Linux CLI knowledge

## Implementation

After connecting via SSH to my VPS, switching to root and installing NGINX, I created a new configuration file in `/etc/nginx/sites-enabled` named `federico.mahfoud.ar` with the following configuration:

```nginx
server {
	listen 80;
	listen [::]:80;
	server_name federico.mahfoud.ar;

	proxy_set_header Host fedemahf.github.io;
	proxy_set_header X-Real-IP $remote_addr;

	location /resume {
		proxy_pass https://fedemahf.github.io/resume/;
	}

	location /blog {
		proxy_pass https://fedemahf.github.io/blog/;
	}

	location / {
		proxy_pass https://fedemahf.github.io/federico.mahfoud.ar/;
	}
}
```

Some details about each directive:
- `listen`: port to listen to
- `server_name`: domain for this specific server config
- `proxy_set_header`: header to send to the proxied server (in this case, GitHub). It's important to send the correct `Host` header, otherwise GitHub will not be able to find your GitHub Page.
- `proxy_pass`: for this specific location block, proxy the request to the corresponding URL

After creating this file, the configuration can be tested using:

```sh
nginx -t
```

If everything is fine, you should be able to reload the NGINX configuration.

```sh
nginx -s reload
```

At this point, your reverse proxy should be working. You need to point your domain (the one you used in the `server_name` directive) to your VPS to finish the setup.

## Post installation (optional)

### SSH security
I recommend taking some steps more for security.

Secure your SSH server in your VPS:
- create a personal SSH key and use it to log into your user
- disable SSH password authentication (`PasswordAuthentication no`) and root login (`PermitRootLogin no`)
- install and configure fail2ban for the ssh daemon

Useful tutorial: [How to Set Up SSH Keys on Debian 11](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-debian-11) by *Jamon Camisso*

### CloudFlare security

In my case, I'm using CloudFlare to proxy all the requests and hide the real location of my NGINX server. If all the requests to my NGINX server are coming from CloudFlare, then I only need to listen to CloudFlare IPs. I used the [Allow CloudFlare only](https://gist.github.com/Manouchehri/cdd4e56db6596e7c3c5a) script by *Manouchehri* to drop all connections on ports 80,443 that aren't coming from CloudFlare.

Also, I used the [`cloudflare-sync-ips.sh` script](https://github.com/ergin/nginx-cloudflare-real-ip) by *ergin* to see the real user IPs in the NGINX access log.