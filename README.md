# Introduction to Virtualization and Distributed Programming - Part 2

## Summary

In this lab a cloud application is built using AWS, and where: A basic Java user registration application runs in a Docker microcontainer on an EC2 machine; within this machine, there is another microcontainer containing the Mongo database, with which the application interacts; In order to access this EC2 machine, an API Gateway was created with Springboot and is deployed on another EC2 machine.

![](imgs/arqui-intro-2.png)

## Prerequisites

* Have an AWS account
* Have Docker installed locally and on the EC2 machine where you are going to deploy the web application

```
sudo yum update -y
sudo yum install docker
sudo service docker start
sudo usermod -a -G docker ec2-user

**Log out of and reconnect to the EC2 machine to apply the changes**
```

* Have Docker installed locally and on the EC2 machine where you are going to deploy the web application

```
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```
```
docker-compose --version
```

***Recommendations:*** 
* Assign an storage greater than 8GB to the EC2 machine where the web application will be deployed; in this case the assigned storage was 16GB.
* Create and assign Elastic IP addresses to your EC2 machines.

## Features

#### Java
```
Java version: 17
```

### Structure

#### Web Application folder: [intro](https://github.com/nduran06/AYGO-Intro-2/tree/master/intro) 

* **Port:** 8090
* **User access:** http://localhost:8090/signup.html

![](imgs/local_signup.png)

#### API Gateway folder: [api.gateway](https://github.com/nduran06/AYGO-Intro-2/tree/master/api.gateway) 

* **Port:** 8080

For future Docker configuration, the folder with the dependencies must be generated in each app, that is why this plugin is added to the POM:
![](imgs/plugin.png)

```
    <build>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-dependency-plugin</artifactId>
				<executions>
					<execution>
						<id>copy-dependencies</id>
						<phase>package</phase>
						<goals>
							<goal>copy-dependencies</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>
```


### Docker-Database Config (AWS)

#### MongoDB container

1. In your web application EC2 machine create a mongo-init.js file, with this content:

```
db.createUser(
        {
            user: "<Database user username>",
            pwd: "<Database user password>",
            roles: [
                {
                    role: "readWrite",
                    db: "<Database name to create>"
                }
            ]
        }
);

```

2. In the same folder where you created the mongo-init.js file, create the docker-compose.yml with this content:

```
services:
    mongodb:
        image: mongo:latest
        container_name: <Docker container name>
        restart: always
        environment:
            MONGO_INITDB_ROOT_USERNAME: <Database user username>
            MONGO_INITDB_ROOT_PASSWORD: <Database user password>
            MONGO_INITDB_DATABASE: <Database name to create>
        ports:
            - 27017:27017
        volumes:
            - ./mongo-init.js:/docker-entrypoint-initdb.d/mongo-init.js:ro

```
3. Create your container with Docker Compose:

```
docker-compose up --build -d mongodb
```

You can try to access the corresponding EC2 IP/DNS with the configured port from your browser. You should be able to see this message:

**From the IP:**

![](imgs/db_access1.png)

**From the DNS:**

![](imgs/db_access2.png)


#### *IMPORTANT!!!:* Don't forget to open the ports you configured, in the inbound rules of your EC2 instance's security group.

#### Code

1. Change the value of the URI attribute for the database connection to the corresponding data for the EC2 machine, in the application.properties file

```
mongodb://nduran06:aygopass123@ec2-54-205-104-132.compute-1.amazonaws.com:27017/?retryWrites=true&w=majority
```
![](imgs/config_uri.png)

This configuration follows these parameters:
[<img src="imgs/uri_config.png">](https://www.mongodb.com/docs/drivers/java/sync/v4.3/fundamentals/connection/connect/)


## Apps-Docker Config (Locally)

### Create the containers with *docker run*

1. Create the *Dockerfile* at the root of each project:

**1. Web Application:** [intro](https://github.com/nduran06/AYGO-Intro-2/tree/master/intro)

```
FROM openjdk:17
WORKDIR /usrapp/bin
ENV PORT=8090
COPY /target/classes /usrapp/bin/classes
COPY /target/dependency /usrapp/bin/dependency
CMD ["java","-cp","./classes:./dependency/*","com.docker.intro.IntroApplication"]

```

**2. API Gateway:** [api.gateway](https://github.com/nduran06/AYGO-Intro-2/tree/master/api.gateway) 

```
FROM openjdk:17
WORKDIR /usrapp/bin
ENV PORT=8080
COPY /target/classes /usrapp/bin/classes
COPY /target/dependency /usrapp/bin/dependency
CMD ["java","-cp","./classes:./dependency/*","com.docker.intro.api.gateway.ApiGatewayApplication"]

```

2. Build the Docker images based on the instructions specified in the Dockerfiles:

**1. Web Application**

```
docker build --tag dockerjavaapp .
```
![](imgs/app_build.png)


**2. API Gateway**

```
docker build --tag dockerapigateway .
```
![](imgs/apig_build.png)


3. With docker run create and start a new container, running the previous images:

**1. Web Application**

```
docker run -d -p 8090:8090 --name containerdockerjavaapp dockerjavaapp
```
![](imgs/app_cont.png)

**2. API Gateway**

```
docker run -d -p 8080:8080 --name containerdockerapigateway dockerapigateway
```
![](imgs/apig_cont.png)


### Create the containers with *docker compose*

1. Create the *docker-compose.yml* file at the root of each project:

**1. Web Application:** [intro](https://github.com/nduran06/AYGO-Intro-2/tree/master/intro)

```
services:
    user_web_app:
        build:
            context: .
            dockerfile: Dockerfile
        container_name: container_user_web_app
        ports:
            - "8090:8090"

```

**2. API Gateway:** [api.gateway](https://github.com/nduran06/AYGO-Intro-2/tree/master/api.gateway) 

```
services:
    apigateway_app:
        build:
            context: .
            dockerfile: Dockerfile
        container_name: container_apigateway
        ports:
            - "8080:8080"

```

2. Build, create, start, and attach the containers for each project:

```
docker-compose up -d
```

**1. Web Application** 

![](imgs/app_compose.png)

**2. API Gateway** 
![](imgs/apig_compose.png)


Using Docker Desktop, you can verify your configuration:


**1. Web Application** 

![](imgs/app_dash.png)

**2. API Gateway** 
![](imgs/apig_dash.png)

It should now be possible to make a post request via the API gateway:

![](imgs/post1.png)
![](imgs/db_record1.png)


### Docker Hub


**1. Web Application** 

1. Repository created: [nduran06/aygo-intro2-webapp](https://hub.docker.com/repository/docker/nduran06/aygo-intro2-webapp/general)

2. Upload the web application docker image to the repository:

> ```docker tag <Docker image name> <Docker Hub repository name>:<tag>```

```
docker tag intro_user_web_app nduran06/aygo-intro2-webapp:v1
```

> ```docker push <Docker Hub repository name>:<tag>```

```
docker push nduran06/aygo-intro2-webapp:v1
```

![](imgs/app_dockhub.png)


3. Go to your EC2 machine intended for the web application, and deploy it:

```
docker run -d -p 8090:8090 --name webappdockeraws nduran06/aygo-intro2-webapp:v1
```

4. When you run ***docker ps -a***, you should see something like this:

![](imgs/conts1.png)

You should now be able to make requests using your EC2 machine's IP/DNS:

![](imgs/post2.png)
![](imgs/db_record2.png)


**2. API Gateway**


***Remember:*** Change the related web application url in the **application.properties** (*userWebApp.path*):


![](imgs/api_code1.png)

Since it is used in the configuration bean:

![](imgs/api_code2.png)

***If required:*** Rebuild the API gateway image:

```
docker compose up -d --no-deps --build apigateway_app
```


1. Repository created: [nduran06/aygo-intro2-apigateway](https://hub.docker.com/repository/docker/nduran06/aygo-intro2-apigateway/general)

2. Upload the web application docker image to the repository:

> ```docker tag <Docker image name> <Docker Hub repository name>:<tag>```

```
docker tag apigateway_apigateway_app:latest nduran06/aygo-intro2-apigateway:v1
```

> ```docker push <Docker Hub repository name>:<tag>```

```
docker push nduran06/aygo-intro2-apigateway:v1
```

![](imgs/api_dockhub.png)

3. Go to your EC2 machine intended for the api gateway, and deploy it:

```
docker run -d -p 8080:8080 --name apigatewaydockeraws nduran06/aygo-intro2-apigateway:v1
```

4. When you run ***docker ps -a***, you should see something like this:

![](imgs/conts2.png)


You should now be able to make requests using your EC2 machine's IP/DNS:

![](imgs/post3.png)
![](imgs/db_record3.png)

## Test run video (Spanish)
[![asciicast](https://github.com/nduran06/AYGO-Intro-Part-2-Introduction-to-Virtualization-and-Distributed-Programming/blob/master/imgs/play_click.png)](https://pruebacorreoescuelaingeduco-my.sharepoint.com/:v:/g/personal/natalia_duran-v_mail_escuelaing_edu_co/EWxWdd_DYWtDvYU96hku4ykBJ-1HPeZmCrb-94GEIMi2Dg?nav=eyJyZWZlcnJhbEluZm8iOnsicmVmZXJyYWxBcHAiOiJPbmVEcml2ZUZvckJ1c2luZXNzIiwicmVmZXJyYWxBcHBQbGF0Zm9ybSI6IldlYiIsInJlZmVycmFsTW9kZSI6InZpZXciLCJyZWZlcnJhbFZpZXciOiJNeUZpbGVzTGlua0NvcHkifX0&e=OkgApM)



