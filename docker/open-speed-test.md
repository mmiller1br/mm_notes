
## Docker-compose for OpenSpeedTest

#docker #docker-compose #openspeedtest #container

This is docker-compose file to create a docker container with the OpenSpeedTest container. Differently from SppedTestTracker, this tool check your internet performance (upload and download) in realtime, but it does NOT track or store old tests. It's just a test on realtime to check your up/down performance at the moment you use it.

I keep both containers on my mini-server, just in case ;)

```
version: '3.3'
services:
    speedtest:
        restart: unless-stopped
        container_name: openspeedtest
        ports:
            - '3200:3000'
            - '3201:3001'
        image: openspeedtest/latest
```
