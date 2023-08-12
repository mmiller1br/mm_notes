
## Docker-compose for Guacamole

#docker #docker-compose #guacamole #container

This is docker-compose file to create a docker container with the Guacamole (https://hub.docker.com/r/jwetzell/guacamole).

I use Guacamole in my HomeLAB in order to centralize management access most of the saervices that I have running. From this interface I cab SSH, VNC, etc to my servers and get remote access instantly.

```
version: "2"
services:
  guacamole:
    image: jwetzell/guacamole
    container_name: guacamole
    volumes:
      - /home/mmiller/docker/guacamole/:/config
    ports:
      - 8083:8080
```

