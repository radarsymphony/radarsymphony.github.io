---
title: Traefik Reverse Proxy
date: 2023-10-27T18:38:41-07:00
type: post
draft: true
showTableOfContents: true
tags:
  - how-to
  - traefik
  - reverse-proxy
  - homelab
  - https
---
# Overview
The following guide outlines the steps to run [Traefik](https://traefik.io/traefik/) with docker as a reverse proxy for your host. This setup enables you to resolve hostnames to particular containers running on the host. With a public domain, you can use [Traefik to request SSL certificates to enable https](https://doc.traefik.io/traefik/https/acme/) for each site.
#### Requirements
- Public domain (I use "example.com" throughout, so update these examples with your domain)
- Docker
- Docker Compose
# DNS Setup
To allow Traefik to request SSL certificates, you will need to generate an API key with your DNS provider and identify the email associated with that account. I use [Cloudflare](https://www.cloudflare.com/) as an example. You will need to adjust for your provider. Supported providers are listed in [Traefik's providers documentation](https://doc.traefik.io/traefik/https/acme/#providers).
#### Email
Login to your DNS provider. The email you entered for login is likely the one associated with the account. To confirm, go to your account profile and identify the primary verified email.

![Cloudflare Profile](/images/posts/cloudflare_profile.png)
#### API Key
1. Create an API key.
2. Give it permission to edit the zone's DNS records.
3. Scope the key to a specific domain/zone (optional, but recommended).

In Cloudflare, this process looks like this:

![Cloudflare API key Generation](/images/posts/cloudflare_apikey.png)

![Cloudflare API key Generation](/images/posts/cloudflare_keypermissions.png)

# Local Setup
Now that you've set up your DNS, it's time to set up Traefik.
1. Create a directory to store the Traefik configuration files: `mkdir -p /srv/apps/traefik/data`
2. Change into to that directory: `cd /srv/apps/traefik/data`
3. Create a `traefik.yml` for configuration and `acme.json` for certificates: `touch traefik.yaml acme.json`
4. Change permissions on the `acme.json` file: `chmod 0600 acme.json`
## Config File
1. From the `data` directory, open the traefik configuration file with an editor: `nano traefik.yml`
2. Paste in the following configuration and update the `[values]` to match your information.

```yaml
global:
  checkNewVersion: true
  sendAnonymousUsage: false  # true by default

api:
  dashboard: false  # Enable if you want a traefik dashboard
  insecure: false  # Don't enable this in production! 

entryPoints:
  web:
    address: :80
# START HTTPS section - remove from here to 'END HTTPS' if not using https/SSL certs
    http: 
      redirections: 
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: :443

# Optional middleware to add a header for redirects on some applications
middlewares:
  sslheader:
    headers:
      customResponseHeaders: 
        X-Forward-Proto: "https"

certificatesResolvers: 
  cert-resolver: # You can call this whatever you'd like, update in compose.yml too
    acme:
      email: [DNS_provider_account_email]
      storage: acme.json
      dnsChallenge:
	    # https://doc.traefik.io/traefik/https/acme/#providers
        provider: [provider_code] 
        resolvers:
          - "1.1.1.1:53" # cloudflare resolvers. Use your DNS providers
          - "1.0.0.1:53"
# END HTTPS
providers:
  docker:
    exposedByDefault: false  # Default is true
  file:
    # watch for dynamic configuration changes
    # https://doc.traefik.io/traefik/providers/file/#provider-configuration
    directory: /etc/traefik # mapped in compose.yml
    watch: true

logs: 
  filePath: /var/log/traefik.log # mapped in compose.yml
  level: DEBUG

```
## Docker
1. Create a docker network called `proxy`: `docker network create proxy` (optionally, specify a subnet, e.g., `--subnet=172.20.0.0/24`)
2. Move up to base directory: `cd /srv/apps/traefik`
3. Paste the following into a `compose.yml` file and update the `[values]` with your information.

```yaml
services:
  traefik:
    image: traefik:latest
    container_name: traefik
    security_opt: 
      - no-new-privileges: true
    environment:
      - TZ=[YOUR_TIMEZONE]
      # Lookup environment variables for your provider
      # https://doc.traefik.io/traefik/https/acme/#providers
      - CF_API_EMAIL=[DNS_EMAIL] 
      - CF_DNS_API_TOKEN=[DNS_API_KEY]
    labels:
      - traefik.enable=true
      # Enable dashboard 
      - traefik.http.routers.traefik-dashboard.service=api@internal
      - traefik.http.routers.traefik-dashboard.entrypoints=websecure
	  - traefik.http.routers.traefik-dashboard.rule=Host(`traefik.local.example.com`) 
	    # UPDATE
      - traefik.http.services.traefik-dashboard.loadbalancer.server.port=8080
      - traefik.http.middlewares.sslheader.headers.customrequestheaders.X-Forwarded-Proto=https
      - traefik.http.routers.reverse-proxy.tls=true
      - traefik.http.routers.reverse-proxy.tls.certresolver=cert-resolver
        #- traefik.http.routers.reverse-proxy.tls.domains[0].main=example.com
        #- traefik.http.routers.reverse-proxy.tls.domains[0].sans=*.example.com
      - traefik.http.routers.reverse-proxy.tls.domains[1].main=vpn.example.com
      - traefik.http.routers.reverse-proxy.tls.domains[1].sans=*.vpn.example.com
      - traefik.http.routers.reverse-proxy.tls.domains[2].main=local.example.com
      - traefik.http.routers.reverse-proxy.tls.domains[2].sans=*.local.example.com
      # append more subdomains to generate certificates, if required
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./data/traefik.yml:/etc/traefik/traefik.yml:ro
      - ./data/acme.json:/acme.json
	  - ./logs/traefik.log:/var/log/trafik.log
	  # potential security issue, consider proxying socket:
      - /var/run/docker.sock:/var/run/docker.sock:ro 
    networks:
      - proxy
    restart: unless-stopped

networks:
  proxy:
    external: true
```

#### Subdomains
In the docker-compose.yml example above, we are listing subdomains to generate certificates for. You can add the top level domain if you want to use it, but I've decided that anything in my homelab I'll refer to with a \*.local.example.com hostname and reserve the base domain for public services.
# Start Traefik
After configuring the public DNS records and local Traefik configuration, you can start up Traefik.
1. Move to Traefik directory: `cd /srv/apps/traefik`
2. Start container: `docker-compose up -d`
3. Watch for any errors in logs: `docker-compose logs -f`
#### Troubleshooting
- Is another service already using ports 80/443 on this machine? 
- Are there any `[placeholder_values]` or "example.com" remaining in the config files? 

# Test Configuration
To test, you can run a nginx container and confirm that, when you access the site in your browser, the page is served over HTTPS and the certificate matches your subdomain.

Edit your DNS records to point requests for "test.local.example.com" to your localhost. 
1. Edit `/etc/hosts` to include `127.0.0.1   test.local.example.com`
2. Test that the hostname resolves to your localhost with `nslookup`.

Create a new compose file (example `test-compose.yml`) with the following content:
```yaml
services:

  nginx-test:
    image: nginx:latest
    container_name: nginx-test
    labels:
      - traefik.enable=true
      - traefik.http.routers.nginx-test.tls=true
      - traefik.http.routers.nginx-test.rule=Host(`test.local.example.com`)
      - traefik.http.routers.nginx-test.service=nginx-test
      - traefik.http.services.nginx-test.loadbalancer.server.port=80
    networks:
      - proxy

networks:
  proxy:
    external: true
```
To run the test:
1. Start the nginx test container: `docker-compose -f test-compose.yml up` (no `-d` to monitor the logs)
2. In a browser, go to "test.local.example.com" 

The default nginx site should load and the url bar should indicate that the connection is secure.
#### Troubleshooting
- Inspect the container logs: `docker-compose -f test-compose.yml logs -f`
- Confirm the "test.local.example.com" hostname resolves to localhost with `nslookup`. If it doesn't, check `/etc/nsswitch.conf` to make try moving `files` is before `resolve` and restarting the `systemd-resolved.service`.

If you get stuck, please reach out by opening an issue in the repo that tracks this website. You can find it here: https://github.com/radarsymphony/radarsymphony.github.io/issues
# Resources
- [Traefik's documentation on providers](https://doc.traefik.io/traefik/https/acme/#providers)
- [Christian Lempa's youtube tutorial on Traefik](https://youtu.be/wLrmmh1eI94?si=IIxc8CtTpPfAIaVg)
- [Techno Tim's youtube tutorial on SSL and Traefik](https://www.youtube.com/watch?v=liV3c9m_OX8&t=171s)
