# Simple 2 Sites on 1 Server Example

In this example will demonstrate the setup for 2 domains pointing to the same server with a basic configuration. The key elements are:
* Creating a docker network outside the compose files
* All containers have a fixed name and join the same docker network (or atleast the ones handled by the reverse proxy)
* Within the nginx.conf (<yourdomain.conf>) files use the service container name as location
* Only the reverse proxy should expose the port 80 and 443
* You can split services in the docker-compose-services.yml file into seperate service files per page


## Prerequisit:

- Docker
- Docker-Compose
- 2 DNS entries pointing to your Server
- ssl certificates for your domain
- Port 80 and 443 opened in firewall

## Steps:

* In Terminal create a docker network to decouple the network from one docker-compose file.

``` bash
 docker network create testNet
```

* Replace the line 4 and the ssl_certificates with your registered website domain.
```sh
server {
  listen 80;

  server_name yourDomain.com;
  
    # ship the acme-challenge to the shared mounted folder 
    # between the certbot and the reverseproxy container
    location /.well-known/acme-challenge/ {
      root /var/www/certbot;
    }

    # redirect http request to https
    location / {
      return 301 https://$host$request_uri;
    }

    error_page   500 502 503 504  /50x.html;

    location = /50x.html {
      root   /usr/share/nginx/html;
    }
}

server {
  listen 443 ssl;
  server_name yourDomain.com;

  # previously aquirred certificates mounted in the reverse proxy container
  # replace the yourDomain.com with your actuall domain
  ssl_certificate /etc/letsencrypt/live/yourDomain.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/yourDomain.com/privkey.pem;
  include /etc/letsencrypt/options-ssl-nginx.conf;
  ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

  location / {
    proxy_pass http://apache/;
    proxy_buffering off;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```
Rename the files in the nginx folder corresponding to your sites, although that is optional.

* the docker-compose-proxy.yml file
```bash
version: '3.8'
services:

  reverse-proxy:
    container_name: 'nginx-service'
    build:
      context: .
      dockerfile: nginx.Dockerfile
    ports:
      - 80:80
      - 443:433
    restart: unless-stopped
    network_mode: testNet
    volumes:
      # volume to share the current certificate (read-only)
      - ./certbot/conf:/etc/letsencrypt:ro
      # volume to share the acme-challenge
      - ./certbot/www:/var/www/certbot

  certbot:
    image: certbot/certbot
    restart: unless-stopped
    network_mode: testNet
    container_name: 'certbot'
    volumes:
      # volume to share the current certificate
      - ./certbot/conf:/etc/letsencrypt
      # volume to share the acme-challenge
      - ./certbot/www:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
```

*  start the services with docker-compose

``` bash
  docker-compose -f docker-compose-services.yml up -d
  docker-compose -f docker-compose-proxy.yml up -d
```

if everything was correct 
``` bash
  docker ps 
```
lists 4 running container and your website should be reachable with a secure https connection.