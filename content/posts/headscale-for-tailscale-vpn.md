---
title: Heads or Tailscale VPN
date: 2023-10-31T15:01:39-07:00
draft: false
type: post
showTableOfContents: true
tags:
  - vpn
  - tailscale
  - headscale
  - opensource
  - server
  - homelab
  - network
---
# Overview
This guide outlines how to set up [Headscale](https://headscale.net/) running as Docker container behind a reverse proxy (Traefik). It uses a free ubuntu VPS from the Oracle Cloud Free Tier, but any linux-based host with public IP and about ~1GB of memory should work for small Home Lab setups. 

Headscale is an opensource reverse-engineered implementation of the closed source Tailscale coordination server. There are many advantages to using the original Tailscale coordination server, such as a convenient admin panel and multiple tailnets. However, I am on a quest to explore opensource and privacy-focused software, I've decided to set up Headscale as my Tailscale coordination server. 

Setting up Headscale behind a reverse-proxy is not something that the maintainers support or use themselves, but it _is_ a feature that is often [requested by community members](https://github.com/juanfont/headscale/issues/527). I wanted to see if I could identify a way to configure Headscale behind Traefik as a reverse proxy. The following is my working prototype. 
#### Prerequisites
- Linux VPS with Public IP and ~1GB of memory
- Traefik running in docker container ([Review this guide](/posts/traefik-reverse-proxy))
- Public domain
- [Must read this Headscale documentation](https://github.com/juanfont/headscale/blob/main/docs/reverse-proxy.md)
# VPS Setup
1. Create a free VPS with Oracle Cloud.
2. Point a hostname to the public IP of your VPS (e.g. `headscale.example.com`).
3. Update packages and install docker and docker-compose.
4. Set up a container running Traefik as described in [this guide](/posts/traefik-reverse-proxy). 
# Headscale Setup
1. Login to the VPS with SSH.
2. Create a directory for Headscale: `mkdir /srv/apps/headscale`
3. Change to that directory: `cd /srv/apps/headscale`
4. Create a compose.yml file and paste the following into, updating values as required:

```yaml
services:
  headscale:
    image: headscale/headscale:latest
    container_name: headscale
    hostname: headscale
    labels:
      - traefik.enable=true
      - traefik.http.routers.headscale-example-com.tls=true
      - traefik.http.routers.headscale-example-com.rule=Host(`headscale.example.com`)
      - traefik.http.routers.headscale-example-com.service=headscale-example-com
      - traefik.http.services.headscale-example-com.loadbalancer.server.port=8080
    ports:
      - 3478:3478/udp # did not need to open in Oracle VPC firewall
    volumes:
      - ./data/headscale:/var/lib/headscale
      - ./data/config:/etc/headscale
    networks:
      - proxy
    restart: unless-stopped
    command: "headscale serve"

networks:
  proxy:
    external: true
```

5. Create the config directory: `mkdir -p data/config`
6. Create the sqlite db: `touch data/config/db.sqlite`
7. Create and edit the config file: `vi data/config/config.yaml` (**.yaml** not .yml seems to matter) 
8. Paste in the configuration below, updating values as required:

```yaml
---
server_url: https://headscale.example.com
listen_addr: 0.0.0.0:8080
metrics_listen_addr: 0.0.0.0:9090
grpc_listen_addr: 0.0.0.0:50443
grpc_allow_insecure: false
private_key_path: /var/lib/headscale/private.key
noise:
  private_key_path: /var/lib/headscale/noise_private.key
ip_prefixes:
  - fd7a:115c:a1e0::/48
  - 100.64.0.0/10
derp:
  server:
    enabled: false
    region_id: 999
    region_code: "headscale"
    region_name: "Headscale Embedded DERP"
    stun_listen_addr: "0.0.0.0:3478"
  urls:
    - https://controlplane.tailscale.com/derpmap/default
  paths: []
  auto_update_enabled: true
  update_frequency: 24h
disable_check_updates: false
ephemeral_node_inactivity_timeout: 30m
node_update_check_interval: 10s
db_type: sqlite3
db_path: /etc/headscale/db.sqlite
acme_url: https://acme-v02.api.letsencrypt.org/directory
tls_letsencrypt_challenge_type: HTTP-01
tls_letsencrypt_listen: ":http"
tls_cert_path: ""
tls_key_path: ""
log:
  format: text
  level: info
acl_policy_path: ""
dns_config:
  override_local_dns: true
  nameservers:
    - 1.1.1.1
    - 1.0.0.1
  domains: []
  magic_dns: true
  base_domain: example.com
unix_socket: /var/run/headscale/headscale.sock
unix_socket_permission: "0770"
logtail:
  enabled: false
randomize_client_port: false
```

**Note:** [The full Headscale configuration](https://github.com/juanfont/headscale/blob/main/config-example.yaml) with comments and default values can be found on their Github. This original will likely be useful when debugging.

# Connecting Tailscale Nodes
This process is fairly straight forward and based on the [Headscale documentation](https://headscale.net/running-headscale-linux/#using-headscale).
The only caveat is that you need to prefix the `tailscale` commands with `docker exec [tailscale_container]...`
# Toubleshooting
- Headscale makes a [larger debug-focus version of the headscale image](https://headscale.net/running-headscale-container/#debugging-headscale-running-in-docker). This is useful to include more tools within the container to check connections and troubleshoot.

It took me a little bit of testing to arrive at the combination of URL and listening address. In the end, I found that the following worked well. Initially I was trying to use port 443 (which some other tutorials showed) and received errors like `Client sent an HTTP request to an HTTPS server.` 
```yaml
server_url: https://headscale.example.com
listen_addr: 0.0.0.0:8080
```

It was important to leave the `tls_cert_path` and `tls_key_path` empty, as discussed on the [Headscale Github](https://github.com/juanfont/headscale/blob/main/docs/tls.md#bring-your-own-certificate).
```yaml
acme_url: https://acme-v02.api.letsencrypt.org/directory
tls_letsencrypt_challenge_type: HTTP-01
tls_letsencrypt_listen: ":http"
tls_cert_path: ""
tls_key_path: ""
```

If you have suggestions or would like another pair of eyes to toubleshoot, please reach out by opening an issue in the repo that tracks this website. You can find it here: 

[https://github.com/radarsymphony/radarsymphony.github.io/issues](https://github.com/radarsymphony/radarsymphony.github.io/issues)
# Resources
- [Headscale discussion on reverse-proxies](https://github.com/juanfont/headscale/blob/main/docs/reverse-proxy.md)
- [Headscale documentation on running as container](https://headscale.net/running-headscale-container/)
- [Another blog outlining a similar process](https://techoverflow.net/2022/12/28/headscale-docker-compose-config-for-traefik-https-reverse-proxy/)
