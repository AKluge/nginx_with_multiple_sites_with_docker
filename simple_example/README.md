# Simple 2 Sites on 1 Server Example

In this example will demonstrate the setup for 2 domains pointing to the same server with a basic configuration. The key elements are:
* Creating a docker network outside the compose files
* All contain have a fixed name and join the same docker network
* Within the nginx.conf (<yourdomain.conf>) files use the service container name as location
* Only the reverse proxy should expose the port 80


## Prerequisit:

- Docker
- Docker-Compose
- 2 DNS entries pointing to your Server
- Port 80 opened in firewall

## Steps:

* In Terminal create a docker network to decouple the network from one docker-compose file.

``` bash
 docker network create testNet
```

* Replace the line 4 with your registered website domain.
```sh
server {
  listen 80;

  # replace with your site adress and keep the ; at the line end
  server_name yourDomain.com;

  location / {
    # referenz the docker container by name
    proxy_pass http://apache/;
    proxy_buffering off;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

Rename the files in the nginx folder corresponding to your sites, although that is optional.

*  the services with docker-compose

``` bash
 
 docker-compose -f docker-compose-services.yml up -d
 docker-compose -f docker-compose-proxy.yml up -d
```

if everything was correct 
``` bash
docker ps 
```
lists 3 running container and your website should be reachable.