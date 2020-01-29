
Difference Btn VMs and Containers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
--> Containers and Vms are two ways to deploy multiple, isolated services on a single platform.

VMs
--> A VM will run any software that runs on the bare metal hardware while providing isolation from the real hardware. 
--> Type one hypervisors run on the bare metal while type 2s have an underlying OS

Containers
--> Containers also provide a way to isolate apps and provide a virtual platform for apps to run.

NOTE: 
- Container system requires an underlying OS that provides basic services to all the containerized apps using virtual memory support for isolation.
- A hypervisor on the other hand runs VMs that have their own OS using hardware VM support.

Traditional VMs Characteristics
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- Robust
- Monolithic
- Slow to Boot
- Heavy

Containers Characteristics
~~~~~~~~~~~~~~~~~~~~~~~~~~~
- Lean
- Portable
- Light-weight
- Efficient
- Isolated

CONTAINERS
- Docker is the leading container
- Docker eliminates the interaction between the HOST OS and the hypervisor improving performance.

Where Containers are useful.
- You can do pretty anything with them
- CI builds (Jenkins, Bamboo, etc)
- Deploying to production and scaling

INSTALLATION.
- visit the Docker site and follow the instructions

Docker Architecture.
~~~~~~~~~~~~~~~~~~~~~
Image

Docker Engine
~~~~~~~~~~~~~~
- Builds the images and runs the containers

Basic Commands
```````````````
- Dockerfile is utilized to build and run images.

$ docker build
- builds the image

$ docker run
- runs the image

$ docker logs
- shows the logs from the container

$ docker ps 
- show the docker processes

NOTE:
- Docker has a layered file system
- Each later is represented by a harsh value. 
- When rebuilding the engine doesn't rebuild everything from scratch especially when caching has been enabled. Layers that haven't been affected are retained.
- Docker has a caching mechanism that keeps track of changes.


Docker Hub
~~~~~~~~~~~
- Is a repository for docker images

Basic Commands
````````````````
> docker push
-pushes the image to the repo

> docker pull
- pull the image with the latest tag from the repo

NOTE:
- You can have either public or private repo on docker hub.
- Docker can run other lunux or windows utilities e.g the linux bash
$ docker run -it ubuntu /bin/bash


Docker Compose
~~~~~~~~~~~~~~~
- Describes the components of an app.
- Yaml file is used to describe the components
- Docker compose wires together different images of your choice and need and builds/rebuilds them together. 

Basic Commands:
~~~~~~~~~~~~~~~
$ docker-compose up
- Will start all the containers

$ docker-compose build
- rebuilds your image

$ docker-compose stop
- Stops the containers

Docker Machine
~~~~~~~~~~~~~~~~
- Provisions and manages docker hosts

Basic Commands
```````````````
$ Docker-machine create
- create a new docker host

$ docker-compose ssh
- connect to the host using ssh

$ docker-compose rm 
- destroy the host

$ docker-compose env 
-sets environment variables for your client to connect to the host

ORCHESTRATION
- Can be accomplished by use of either of the following:
--> MESOS - Cluster manager
--> Docker - Swarm
--> Kubernetes

TERMINOLOGIES
1. Image
- Is an executable package that includes everything needed to run an app i.e, the code, runtime, libraries, environment variables, and config files
- Simply we can say it is a container blueprint.

2. Container 
- It is a runtime instance of an image- what the image becomes in memory when executes i.e an image with a state or user process.
