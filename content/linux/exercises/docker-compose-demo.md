# Instructor Demo: Docker Compose

In this demo, we'll illustrate:

 - Starting an app defined in a docker compose file
 - Inter-service communication using DNS resolution of service names

## Exploring the Compose File

1.  Please download the DockerCoins app from Github and change directory to ~/orchestration-workshop/dockercoins.

    ```bash
    [centos@node-0 ~]$ git clone -b ee3.0 \
        https://github.com/docker-training/orchestration-workshop.git
    [centos@node-0 ~]$ cd ~/orchestration-workshop/dockercoins
    ```

2.  Let's take a quick look at our Compose file for Dockercoins:

    ```yaml
    version: "3.1"

    services:
    rng:
        image: training/dockercoins-rng:1.0
        networks:
        - dockercoins
        ports:
        - "8001:80"

    hasher:
        image: training/dockercoins-hasher:1.0
        networks:
        - dockercoins
        ports:
        - "8002:80"

    webui:
        image: training/dockercoins-webui:1.0
        networks:
        - dockercoins
        ports:
        - "8000:80"

    redis:
        image: redis
        networks:
        - dockercoins

    worker:
        image: training/dockercoins-worker:1.0
        networks:
        - dockercoins

    networks:
        dockercoins:
    ```

    This Compose file contains 5 services, along with a bridge network.

3.  When we start the app, we will see the service images getting downloaded one at a time:

    ```bash
    [centos@node-0 dockercoins]$ docker-compose up -d
    ```

4.  After starting, the images required for this app have been downloaded:

    ```bash
    [centos@node-0 dockercoins]$ docker image ls | grep "dockercoins"
    ```

5.  Make sure the services are up and running, as is the dedicated network:

    ```bash
    [centos@node-0 dockercoins]$ docker-compose ps
    [centos@node-0 dockercoins]$ docker network ls
    ```

6.  If everyting is up, visit your app at `<node-0 public IP>:8000` to see Dockercoins in action.

## Communicating Between Containers

1.  In this section, we'll demonstrate that containers created as part of a service in a Compose file are able to communicate with containers belonging to other services using just their service names. Let's start by listing our DockerCoins containers:

    ```bash
    [centos@node-0 dockercoins]$ docker container ls | grep 'dockercoins'
    ```

2.  Now, connect into one container; let's pick `webui`:

    ```bash
    [centos@node-0 dockercoins]$ docker container exec -it <Container ID> bash
    ```

3.  From within the container, ping `rng` by name:

    ```bash
    [root@<Container ID>]# ping rng
    ```

    Logs should be outputted resembling this:

    ```bash
    PING rng (172.18.0.5) 56(84) bytes of data.
    64 bytes from dockercoins_rng_1... (172.18.0.5): icmp_seq=1 ttl=64 time=0.108 ms
    64 bytes from dockercoins_rng_1... (172.18.0.5): icmp_seq=2 ttl=64 time=0.049 ms
    64 bytes from dockercoins_rng_1... (172.18.0.5): icmp_seq=3 ttl=64 time=0.073 ms
    64 bytes from dockercoins_rng_1... (172.18.0.5): icmp_seq=4 ttl=64 time=0.067 ms
    64 bytes from dockercoins_rng_1... (172.18.0.5): icmp_seq=5 ttl=64 time=0.057 ms
    64 bytes from dockercoins_rng_1... (172.18.0.5): icmp_seq=6 ttl=64 time=0.074 ms
    64 bytes from dockercoins_rng_1... (172.18.0.5): icmp_seq=7 ttl=64 time=0.052 ms
    64 bytes from dockercoins_rng_1... (172.18.0.5): icmp_seq=8 ttl=64 time=0.057 ms
    64 bytes from dockercoins_rng_1... (172.18.0.5): icmp_seq=9 ttl=64 time=0.080 ms
    ```

    Use `CTRL+C` to terminate the ping. DNS lookup for the services in DockerCoins works because they are all attached to the user-defined `dockercoins` network.

4.  After exiting this container, let's navigate to the `worker` folder and take a look at a section of `worker.py`:

    ```bash
    [centos@node-0 dockercoins]$ cd worker
    [centos@node-0 worker]$ cat worker.py

    import logging
    import os
    from redis import Redis
    import requests
    import time

    DEBUG = os.environ.get("DEBUG", "").lower().startswith("y")

    log = logging.getLogger(__name__)
    if DEBUG:
        logging.basicConfig(level=logging.DEBUG)
    else:
        logging.basicConfig(level=logging.INFO)
        logging.getLogger("requests").setLevel(logging.WARNING)

    redis = Redis("redis")

    def get_random_bytes():
        r = requests.get("http://rng/32")
        return r.content

    def hash_bytes(data):
        r = requests.post("http://hasher/",
                        data=data,
                        headers={"Content-Type": "application/octet-stream"})
        hex_hash = r.text
        return hex_hash
    ```

    As we can see in the last two stanzas, we can direct traffic to a service via a DNS name that exactly matches the service name defined in the docker compose file.

5.  Shut down Dockercoins and clean up its resources:

    ```bash
    [centos@node-0 dockercoins]$ docker-compose down
    ```

## Conclusion

In this exercise, we stood up an application using Docker Compose. The most important new idea here is the notion of Docker Services, which are collections of identically configured containers. Docker Service names are resolvable by DNS, so that we can write application logic designed to communicate service to service; all service discovery and load balancing between your application's services is abstracted away and handled by Docker.
