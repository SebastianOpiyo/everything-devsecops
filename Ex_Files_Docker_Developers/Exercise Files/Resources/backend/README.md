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
