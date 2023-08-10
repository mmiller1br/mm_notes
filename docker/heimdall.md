
## Docker-compose for Heimdall

#docker #docker-compose #heimdall #container

This is docker-compose file to create a docker container with the Heimdall Dashboard.

I use this docker to create a WEB Dashboard for all my APPs, Continers, Services etc in my HomeLAB. This is not my primary dashboard, but I'm still using it to organize my services and access them faster using a nice dashboard.


```
---
version: "2.1"
services:
  heimdall:
    image: lscr.io/linuxserver/heimdall:latest
    container_name: heimdall
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /home/mmiller/docker/heimdall:/config
    ports:
      - 7480:80
      - 7443:443
    restart: unless-stopped
```
