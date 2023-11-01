---
title: Tailscale Mesh VPN
date: 2023-10-31T13:47:56-07:00
draft: false
type: post
showTableOfContents: true
tags:
  - VPN
  - homelab
  - tailscale
  - network
  - docker
---
# Overview
[Tailscale](https://tailscale.com/) is an easy to configure mesh VPN. It uses [NAT traversal](https://tailscale.com/blog/how-nat-traversal-works/) to connect peers to each other. This article will outline the steps to set up Tailscale running in Docker as if it were running on the docker host directly (without SSH over the VPN - yet). This approach enables managing the Tailscale connection as you would any other docker service and creates a portable and deploy-able compose.yml to run on other systems. 
# Tailscale Account
Tailscale makes use of a control panel to orchestrate and connect nodes. In order to connect your devices together, you need to create an account with Tailscale. 

> In [this article](/posts/headscale-for-tailscale-vpn), I modify the VPN outlined here to move away from the closed source Tailscale control server to the opensource Headscale control server. 

1. Create an account with Tailscale [login.tailscale.com](https://login.tailscale.com/)
2. Go to **Settings** > **Personal Settings** > **Keys** 
3. Click **Generate auth key**, turn on **Reusable** 
4. Click **Generate key** and save the value.
# Docker Compose
1. Create a directory to host this service: `mkdir /srv/apps/tailscale`
2. Change into that directory: `cd /srv/apps/tailscale`
3. Create a compose.yml file: `touch compose.yml`
4. Open the file up with an editor and paste in the following:

```yaml
services:
  tailscale:
    image: tailscale/tailscale
    container_name: tailscale
    network_mode: "host"
    environment:
      - TZ=[YOUR_TIMEZONE]
      - TS_HOSTNAME=${HOSTNAME}-tailscale
      - TS_AUTHKEY=[TS_AUTHKEY]
      - TS_STATE_DIR=/tailscale/state
      - TS_SOCKET=/var/run/tailscale/tailscaled.sock
      - TS_TAILSCALED_EXTRA_ARGS=--tun=tailscale0 # important 
      - TS_DEBUG_FIREWALL_MODE=nftables
      - TS_NO_LOGS_NO_SUPPORT=true
    volumes:
      - ./data/tailscale/state:/tailscale/state
      - /dev/net/tun:/dev/net/tun # important
    cap_add:
      - net_admin
      - sys_module
    restart: unless-stopped
```

5. Update the TS_AUTHKEY that you generated in the account.
# Start Tailscale
1. While logged in to the Tailscale control panel, go to [the machines tab](https://login.tailscale.com/admin/machines).
2. In your terminal at the Tailscale directory, run: `docker-compose up -d`
3. Refresh the machines tab and you should see a new node appear.
# Testing
A quick way to test if the VPN is working is to install the [Tailscale app on a mobile device](https://tailscale.com/downloads). You should be able to see both devices registered in your machine's tab in Tailscale.
# Troubleshooting
Start simple and progress in small steps. When I set this up, I started with installing the Tailscale binary on two servers (no Docker) exactly like the Tailscale documentation described. This approach allowed me to just see if the two hosts would be able to connect as intended before adding another layer of complexity. I then slowly transitioned one server to use Docker, then the other. 

Oracle Cloud Free Tier allows the provision of a small VPS within minutes that can be useful for having another remote (plain/fresh) node to set up Tailscale on and test iterations. 

I frequently ran `docker exec tailscale tailscale [status|ping <tailscale-hostname>]` to see if the current node was registering other nodes and to gain insight into things like whether it was using IPv4 or IPv6.

I initially had issues with [iptables vs nftables](https://github.com/tailscale/tailscale/issues/5621) and found that for my devices I had to set `TS_DEBUG_FIREWALL_MODE=nftables`. If you encounter issues, you can try removing this value.

To run Tailscale in Docker as if it were running on the host posed some challenges. The nodes were registering but not fully connecting to each other until I added `TS_TAILSCALED_EXTRA_ARGS=--tun=tailscale0`. By default, the [Tailscale Docker image enables "userspace-networking"](https://tailscale.com/kb/1282/docker/?q=docker#ts_userspace) which doesn't seem to work in this setup. 

Depending on your firewall, you may find pointers in the [Tailscale firewall documentation](https://tailscale.com/kb/1181/firewalls/).

If you have suggestions or would like another pair of eyes to toubleshoot, please reach out by opening an issue in the repo that tracks this website. You can find it here: 

[https://github.com/radarsymphony/radarsymphony.github.io/issues](https://github.com/radarsymphony/radarsymphony.github.io/issues)
# Resources
- [Tailscale Terminology and Concepts reference](https://tailscale.com/kb/1155/terminology-and-concepts/)
- https://www.wundertech.net/how-to-set-up-tailscale-on-docker/
- https://docs.ibracorp.io/tailscale/tailscale/docker-compose
- [Similar approach but Tailscale within each docker container](https://tailscale.dev/blog/docker-mod-tailscale)
- [Setting up Headscale to run in Docker behind Reverse Proxy](/posts/headscale-for-tailscale-vpn)