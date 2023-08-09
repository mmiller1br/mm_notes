
This is docker-compose file to create a docker container with the datadog-agent.

In my homeLAB, I have a Ubuntu Server running in a VM on my PROXMOX with a bunch of docker containers. One of these containers is the datadog-agent that reads all the information from this server and send it all to the Cloud.

Once uploaded to the cloud, you can create your Dashboards with information that are more relevant to your needs. Cheers !

```
version: '3'

services:
  datadog:
    image: gcr.io/datadoghq/agent:7
    container_name: datadog-agent
    restart: always
    environment:
      - DD_API_KEY=3de5cf93e70fc8ff91f2629c25aeb47a
      - DD_SITE=datadoghq.com
      - DD_TAGS="machineID:srv-docker"
      - DD_APM_ENABLED=true
      - DD_LOGS_ENABLED=true
      - DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL=true
      - DD_DOGSTATSD_NON_LOCAL_TRAFFIC=true
      - DD_CONTAINER_EXCLUDE=image:gcr.io/datadoghq/agent*
      - DD_CONTAINER_EXCLUDE_METRICS=image:gcr.io/datadoghq/agent*
      - DD_CONTAINER_EXCLUDE_LOGS=image:gcr.io/datadoghq/agent*
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /proc/:/host/proc/:ro
      - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro
        #- /opt/datadog-agent/run:/opt/datadog-agent/run:rw
```

