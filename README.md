# Docker
Docker-Tuts

# Docker Overview
- Makes it easy for developers to set up a new project
- Especially if the devs have a similar project setup across different OS systems
- Makes it trivial especially especially if the setup is the same across different projects & devs


# Main Docker Components
1. Docker Container - where a specific component is located
2. Docker Image - Contains all the info needed to create a new container
3. Volume - is the storage facility for the containers.
4. Networking - Enables separate containers to talk to each other.

# node-mongo-docker
A node and express docker template with Mongo (full-stack)

* APPROACH USED
	> First
		- Create a backend on its own 
		- A backend container will have its own static assets
	> Second 
		- Create a frontend on its own
		- A frontend container will have its own static assets
	> Third
		- A full-stack approach.
		- The dynamic pieces i.e the data/db will be hosted in a volume.

* COMPOSE
 > Is a tool that enables you manage multiple containers and set your entire backend with a single file.

- A simple way to learn docker usage with a simple full-stack app.
