
Containers with Docker
======================

This project showcases how to work with Docker containers when developing applications.
 We will go through several demos:
 1. Use Docker for local development
2. Docker Compose - Run multiple Docker containers
3. Dockerize Nodejs application and push to private Docker registry
4. Deploy Docker application on a server with Docker Compose
5. Persist data with Docker Volumes
6. Create Docker repository on Nexus and push to it
7. Deploy Nexus as Docker container

By using a very simple UI app using JavaScript and Node.js in the backend. In order to integrate
 this in the database, we will use Docker container of a MondoDB database. To make it easy
 we will also deploy a container og a MongoBD UI (Mongo Express), where we can see the database
 structure and the updates that our app is doing in the database.


#### Technologies used:


* Docker, Node.js, MongoDB, MongoExpress


#### 1. Use Docker for local development


To setup locally the application manually:


1.1. Pull docker images for the database:  

`$ docker pull mongo`  

`$ docker pull mongo-express`
  

  

 1.2. Set up a docker network for the MongoDB and MongoExpress:  

`$ docker network create mongo-network`

![mongo-network](img/mongo-network.png)  

  

 1.3. Overwrite the username and password variables and run mongo containers.   

`$ docker run -d -p 27017:27017 -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=password --name mongodb --net mongo-network mongo`  

  
`$ docker run -d -p 8081:8081 -e ME_CONFIG_MONGODB_ADMINUSERNAME=admin -e ME_CONFIG_MONGODB_ADMINPASSWORD=password --net mongo-network --name mongo-express -e ME_CONFIG_MONGODB_SERVER=mongodb mongo-express`
  

  
1.4. Then access the mongo UI and create a "user-account" database and a "users" collection.  

![mongo-user](img/mongo-user.png)  

![users-collection](img/users-collection.png)  
  

 We can now use the app UI to update the profile and see the input in the database:
   
![update](img/update.png)
  

#### 2. Docker Compose - Run multiple Docker containers


We can create a docker-compose.yaml file in order to use those commands for running
 multiple containers in a automated way:


![docker-compose](img/docker-compose.png)  
  

You can then run the command `$ docker compose -f docker-compose.yaml up`:
 ![docker-compose-out](img/docker-compose-out.png)  




#### 3. Dockerize Nodejs application and push to private Docker registry


3.1. Write Dockerfile to build a Docker image for a Nodejs application
   
  
A Dockerfile is a blueprint to create Docker images. All commands in it will be applied to
 a container environment, not in the host computer environment. Exception is the ´COPY´ command.
 Our application has a following Dockerfile:


````

        FROM node:13-alpine

        ENV MONGO_DB_USERNAME=admin \
            MONGO_DB_PWD=password

        RUN mkdir -p /home/app

        COPY ./app /home/app

        # set default dir so that next commands executes in /home/app dir
        WORKDIR /home/app

        # will execute npm install in /home/app because of WORKDIR
        RUN npm install

        # no need for /home/app/server.js because of WORKDIR
        CMD ["node", "server.js"]
                                                
````
Then you can build the image by running `$ docker build -t my-app:1.0 .`,
 and check the image created with `$ docker images`. Run your image:


![app-running](img/app-running.png)  
  

3.2. Create private Docker registry on AWS (Amazon ECR)



![repository](img/repository.png)  
  




3.3. Push Docker image to this private repository


After logging in to the private registry (Docker login), we can use the push commands in
 order to successfully:  

 - Tagg the Docker Image in order to rename them with the correct path to the private repository created  

 - Push the Docker Image to AWS ECR repository  
  

![push-commands](img/push-commands.png)  
  

![images-pushed](img/images-pushed.png)  
  



#### 4. Simulate how to deploy Docker application on a server with Docker Compose



 First we change the mongodb server url from localhost to mongodb service name
 in Node Code: `let mongoUrlDockerCompose = "mongodb://admin:password@mongodb";`.  
  

![mongo-client](img/mongo-client.png)  
  

 Then we add our example application to a Docker-compose file and then run it locally with the
 command `$ docker compose -f mongo.yaml up`:  
  

![compose-file](img/compose-file.png)  
  

#### 5. Persist data with Docker Volumes


In order to persist data of a MongoDB container we attach a Docker volume to it. We need to map
 the default database path for MongoDB (/data/db directory) inside a container
 to our volume in the host computer.
 It can be checked by running `$ docker exec -it [mongodb container id] sh; ls /data/db`.
  

![dockercompose-volumes](img/dockercompose-volumes.png)  
  

Then we can restart the container:  
`$ docker compose -f docker-compose.yaml down`
  
`$ docker compose -f docker-compose.yaml up`
  
and create the database to test it.


#### 6. Create Docker repository on Nexus and push to it


6.1. Create Docker hosted repository on Nexus


6.2. Create Docker repository role on Nexus


6.3. Configure Nexus, DigitalOcean Droplet and Docker to be able to push to Docker repository


6.4. Build and Push Docker image to Docker repository on Nexus


#### 7. Deploy Nexus as Docker container


7.1. Create and Configure Droplet


7.2. Set up and run Nexus as a Docker container





----------------------------------------------------


 










 {% include 'includes/footer-about.html' %}





{% endblock content %}


{% block javascripts %}{% endblock javascripts %}

