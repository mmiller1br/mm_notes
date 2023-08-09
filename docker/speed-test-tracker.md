
## Docker-compose for SpeedTestTracker

#docker #docker-compose #speedtest #container

This is docker-compose file to create a docker container with the speedtest Tracker.

I use this docker to test my Internet Speed (from the VM where my docker containers are running) and give me some data about the performance, with historical information.

Simple, but efficient ;)

```
version: '3.3'
services:
    speedtest:
        container_name: speedtest
        image: henrywhitaker3/speedtest-tracker
        ports:
            - 8765:80
        volumes:
            - /home/mmiller/docker/speedtest/:/config
        environment:
            - TZ=America/Toronto
            - PGID=1000
            - PUID=1000
            - OOKLA_EULA_GDPR=true
        logging:
            driver: "json-file"
            options:
                max-file: "10"
                max-size: "100k"
        restart: unless-stopped
```
