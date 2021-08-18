---
layout: post
title: Docker commands
author: Angel Mary Babu
date: 2021-07-18  11:00:00 +0800
categories: [DOCKER]
tags: [devops, sysops,containerization ]

---


# Useful commands in Docker


#### Commands:

>> To build images from Dockerfile

```sh
docker build -t temp-ubuntu .  
```

>> To list docker images

```sh
docker images 
```

>> To remove an images from local registry

```sh
docker rmi temp-ubuntu:version-1.0 
```

>> To list all the containers

```sh
docker ps -a 
```

>> To run a container.

```sh
docker run -d tmp-ubuntu 
```

>> To pause a container

```sh
docker pause container_id 
```

>> To un-suspends all processes in the specified containers

```sh
docker unpause 
```

>> To restart the container

```sh
docker restart container_id 
```

>> To stop a container

```sh
docker stop container_id 
```

>> To remove a container

```sh
docker rm container_id 
```

>> To download an image from the docker hub.

```sh
docker pull ubuntu 
```

>> To list the images 

```sh
docker image list 
```

>> To start the container, if you haven't downloaded the image then the docker will do it at this time.

```sh
docker run ubuntu:latest 
```

>> This command will map to a port using -p option and run in background with the option -d.

```sh
docker run -p 8080:80 -d nginx:latest 
```

>> To remove the container forcefully

```sh
docker container rm -f container_name 
```
>> To remove docker images

```sh
docker image rm nginx:latest 
```



## Lifecycle of  Docker:

- To place a container in the run state, use the run command. You can also restart a container that is already running. When restarting a container, the container receives a termination signal to enable any running processes to shut down gracefully before the container's kernel is terminated.

- A container is considered in a running state until it's either paused, stopped, or killed. A container, however, may also exit from the run state by itself. A container can self-exit when the running process completes, or if the process goes into a fault state.

- To pause a running container, use the pause command. This command suspends all processes in the container.

- To stop a running container, use the stop command. The stop command enables the working process to shut down gracefully by sending it a termination signal. The container's kernel terminates after the process shuts down.

- To send a kill signal if you need to terminate the container, use the kill command. The running process doesn't capture the kill signal, only the container's kernel. This command will forcefully terminate the working process in the container.

- Lastly, to remove containers that are in a stopped state, use the remove command. After removing a container, all data stored in the container gets destroyed.
