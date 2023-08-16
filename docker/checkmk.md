
## Docker-compose for CheckMK

#docker #docker-compose #checkmk #container

This is docker-compose file to create a docker container with the CheckMK Dashboard.

I use this software as one of the monitoring systems in my HomeLAB, mostly for devices that have SNMP available.


```
---
version: '3'

services:
  checkmk:
    container_name: checkmk
    image: checkmk/check-mk-raw:2.2.0-latest
    tmpfs:
      - /opt/omd/sites/cmk/tmp:uid=1000,gid=1000
    volumes:
      - /home/mmiller/docker/checkmk:/omd/sites
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "8085:5000"
    restart: always
```
