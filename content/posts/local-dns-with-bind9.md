---
title: Local DNS with Bind9
date: 2023-10-31T10:30:32-07:00
draft: false
showTableOfContents: true
type: post
tags:
  - dns
  - homelab
  - bind9
  - resolver
  - docker
---
# Overview
This short article outlines the steps to set up a local DNS server using [BIND9](https://bind9.net/) and Docker on a Raspberry Pi. This approach enables you to manage your own zone(s) for local services running in a Home Lab. It may also help cache DNS queries to reduce lookup time for frequently requested resources. I've only tested this approach in protected local networks. 
#### Prerequisites 
- Docker & docker-compose
- Raspberry Pi (optional\*)
- Static IP for Pi (reserved in your DHCP server/router)

\* You could host this on a personal computer. However, you may run into issues with other DNS services running (e.g., `dnsmasq`). One solution might be to change the port Bind is running on. 
# Docker compose file
1. Create a directory to store the BIND9 configuration: `mdkir -p /srv/apps/bind9`
2. Change to that directory: `cd /srv/apps/bind9`
3. Create a docker-compose file:  `touch compose.yml`
4. Edit the compose.yml file add the following, updating values as required:

```yaml
services:
  ns-local-example-com:
    image: ubuntu/bind9:9.18-22.04_beta
    container_name: ns-local-example-com
    ports:
      - "53:53/udp"
      - "53:53/tcp"
      # https://bind9.readthedocs.io/en/v9.18.19/manpages.html#std-iscman-rndc
      - "127.0.0.1:953:953/tcp"
    environment:
      - BIND9_USER=root
      - TZ=[YOUR_TIMEZONE]
    volumes:
      - ./config:/etc/bind
      - ./cache:/var/cache/bind
      - ./records:/var/lib/bind
      - ./logs:/var/log
      - ./session:/run/named
    restart: unless-stopped
```
# Config files
1. In the root directory of your bind9 service, create a config directory: `mkdir config`
2. Create `config/named.conf` and paste in the following, updating values as need be: 
#### named.conf
```c
acl allowed-networks {
    // add all networks that are allowed to use the dns
    172.0.0.0/8;
    192.168.0.0/24;
    192.168.1.0/24;
    127.0.0.0/24;
    // Allow VPN network
    100.0.0.0/24;
};

options {
    directory	"/var/cache/bind";
    pid-file     "/run/named/named.pid";
    forwarders {
        1.1.1.1;
        1.0.0.1;
        8.8.8.8;
        9.9.9.9;
    };
    allow-query { allowed-networks; };
};

zone "local.example.com" IN {
    type master;
    file "/etc/bind/local.example.com.zone"; // Needs to match name of zone file
};
```

3. Create `config/local.example.com.zone` and paste in the following, updating values as required: 
#### zone file
```lisp
$TTL 2d

$ORIGIN local.example.com.

@		IN	SOA	ns.local.example.com. [YOUR_EMAIL]. (
	
				2023103100 	; serial is any 10-digit code - a date is useful
				12h		; refresh time
				15m		; retry
				3w		; expiry
				2h		; minimum ttl
				)

		IN	NS	ns.local.example.com.

ns		IN	A 	[PI_STATIC_IP]

; -- Add DNS below

mypi		IN	A	    192.168.1.10
*		    IN	CNAME	mypi

```

# Start BIND9
1. With the compose.yml and two configurations in place, start up the service: `docker-compose up -d`
2. Check the logs: `docker-compose logs -f`
# Test
Check the new name server is resolving the records you've added:

`nslookup mypi.local.example.com [PI_STATIC_IP]`

Assuming your Pi's static IP is `192.168.1.10`, you should receive back something like:

```
Server:		192.168.1.10
Address:	192.168.1.10#53

test.local.example.com	canonical name = mypi.local.example.com.
Name:	mypi.local.example.com
Address: 192.168.1.10
```

# Final Steps
To make use of this DNS server, you still need to direct your machines to use it for DNS queries. At this point, it should be running and is able resolve request when asked directly, but your other devices don't know where to find it nor when to use it. Directing devices to use this DNS server is very device-specific. However, below are some steps to try.
#### Router
One way to have devices on your local network is to configure your router to advertise it as a DNS server. To list the BIND9 service for all devices using your router as a gateway, you need to:
1. Login to your router. Often entering `192.168.1.1` in your browser will bring up the login screen.
2. Research your router and find where to specify what DNS servers to use.
3. Add the static IP where your BIND9 service is running.
4. Add a fallback server such as `1.1.1.1`
5. Save.

#### Update each device
Mobiles and laptops tend to have a setting where you can specify which DNS servers to use. 
#### Systemd resolver & NS Switch
I will likely write another post covering this option in more detail.

# Troubleshooting
- Patience. Sometimes DNS takes time to update and propagate. Restarting devices can sometimes help to ensure you're working from a baseline and that the device has requested a new DHCP lease, etc.
- Make sure port 53 (or whatever port you chose) is open on the Raspberry Pi. 
- Read the BIND9 documentation, for example: https://bind9.readthedocs.io/en/latest/chapter3.html#localhost-zone-file

# Resources
- [BIND9 named.conf documentation](https://bind9.readthedocs.io/en/latest/reference.html#named-conf)
- [BIND9 zone file documentation](https://bind9.readthedocs.io/en/latest/chapter3.html)
- [Christian Lempa's youtube on BIND9](https://youtu.be/syzwLwE3Xq4?si=3psNWIJOCqKHozIP)