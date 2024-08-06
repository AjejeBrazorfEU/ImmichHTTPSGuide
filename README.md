# Immich HTTPS setup guide on Raspberry Pi

In this guide we will go through the steps to setup HTTPS on an Immich server. Note that the method used is not directly related to Immich, but rather a general guide on how to setup HTTPS on a server.
In my case, I'm running Immich on a Raspberry Pi 4 with 8 GB of RAM, but the steps should be similar for other servers.

To make the connection to the server secure, we will use [traefik](https://github.com/traefik/traefik) as a reverse proxy and [Let's Encrypt](https://letsencrypt.org/) to generate the SSL certificate.

## Prerequisites

- Domain name (I'm using a free DNS service called [DuckDNS](https://www.duckdns.org/))
- Server running Immich (I'm using a Raspberry Pi 4 with 8 GB of RAM)
- Router with port forwarding capabilities
- Computer with SSH capabilities to connect to the server

## Setup

### 1. Domain name

First, we need to create an account on [DuckDNS](https://www.duckdns.org/) and create a domain name. Note that you have to do this from your local network, in this way DuckDNS will automatically take your public IP address and associate it to your newly created domain name (from now on, we'll call it `yourdomain.duckdns.org`).

By using DuckDNS, we also have access to all the subdomains we want. In this guide, we'll use `traefik.yourdomain.duckdns.org` as the domain name for the Traefik dashboard and `immich.yourdomain.duckdns.org` as the domain name for Immich.

### 2. Port forwarding

The second step is to configure the router to forward the port 443 ([Well known port for HTTPS](https://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers?oldformat=true)) to the server. This is necessary to allow the server to receive HTTPS requests from the internet. This is necessary if you're hosting the server at home, behind a router. If you're hosting it at a cloud provider, you can skip this step.

First find the IP address of the server by running the following command:

```bash
hostname -I
```

Then, access the router configuration page and find the port forwarding section. Add a new rule to forward the port 443 to the IP address of the server. This will allow the server to receive HTTPS requests from the internet.
If you need more help, you can check the [portforward.com](https://portforward.com/) website.

### 3. Docker network

To allow the services to communicate with each other, we need to create a Docker network. To do this, run the following command:

```bash
docker network create traefik-network
```

In this way, all the services will be able to communicate with each other and traefik will be able to access them.

### 4. Traefik configuration

Then, we need to create a `traefik` folder in the root of the server.

```bash
mkdir traefik
cd traefik
```

Inside this folder, create a `docker-compose.yml` file with the following content:

```yaml
# traefik/docker-compose.yml
version: "3.8"

services:
  traefik:
    image: "traefik:v3.0"
    container_name: "reverse_proxy"
    command:
      - "--api.dashboard=true" # Enable Traefik dashboard
      - "--providers.docker=true"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email=${EMAIL}"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
      - "--entrypoints.web.address=:8080"
      - "--entrypoints.websecure.http.tls=true"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
    ports:
      - "8080:8080"
      - "443:443"
    networks:
      - traefik-network
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./letsencrypt:/letsencrypt"
    labels:
      - "traefik.http.routers.api.rule=Host(`${HOSTNAME}`)"
      - "traefik.http.routers.api.service=api@internal"
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.routers.api.tls.certresolver=myresolver"
      - "traefik.http.middlewares.auth.basicauth.users=admin:hashedpassword" # Replace with your own hashed credentials
      - "traefik.http.routers.api.middlewares=auth@docker"
    restart: always

networks:
  traefik-network:
    external: true
```

Also create a `.env` file with the following content:

```bash
# traefik/.env
EMAIL=your-email@example.com
HOSTNAME=traefik.yourdomain.duckdns.org
```

To run this configuration, execute the following commands:

```bash
docker-compose up -d
```

After this, you should be able to access the Traefik dashboard by going to `https://traefik.yourdomain.duckdns.org`. If some error occurs, you can check the logs by running the following command:

```bash
docker logs -f reverse_proxy
```

If this works we're almost done! :tada:

### 5. Immich configuration

Now we need to configure Immich to use the Traefik reverse proxy. To do this, we need to create a `immich` folder in the root of the server.

```bash
mkdir immich
cd immich
```

Inside this folder, create a `docker-compose.yml` file with the default Immich configuration that you can download from the [official repository](https://immich.app/docs/install/docker-compose). This file needs to be modified to use the Traefik reverse proxy. Here is an example of how it should look like:

```yaml
# immich/docker-compose.yml
services:
  immich-server:
    ...
    networks:
      - traefik-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.immich-server.rule=Host(`${HOSTNAME}`)"
      - "traefik.http.routers.immich-server.entrypoints=websecure"
      - "traefik.http.routers.immich-server.tls.certresolver=myresolver"

  immich-machine-learning:
    ...
    networks:
      - traefik-network
    labels:
      - "traefik.enable=false"  # Do not expose via traefik

  redis:
    ...
    networks:
      - traefik-network
    labels:
      - "traefik.enable=false"  # Do not expose via traefik

  database:
    ...
    networks:
      - traefik-network
    labels:
      - "traefik.enable=false"  # Do not expose via traefik

networks:
  traefik-network:
    external: true
```

Also create a `.env` file with the following content:

```bash
# immich/.env
...
HOSTNAME=immich.yourdomain.duckdns.org
```

To run this configuration, execute the following commands:

```bash
docker-compose up -d
```

After this, you should be able to access Immich by going to `https://immich.yourdomain.duckdns.org`.

You've made it!!! :tada: :tada: :tada:
