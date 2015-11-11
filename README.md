# Advanced-Docker


1) **File IO**: You want to create a container for a legacy application. You succeed, but you need access to a file that the legacy app creates.

* Create a container that runs a command that outputs to a file.
* Use socat to map file access to read file container and expose over port 9001 (hint can use SYSTEM + cat).
* Use a linked container that access that file over network. The linked container can just use a command such as curl to access data from other container.

######The Dockerfile for the first container:

        FROM    alpine:3.2
        
        RUN apk update && \
            apk add socat && \
            rm -r /var/cache/
        
        COPY . /src
        RUN cd /src;
        
        CMD socat TCP4-LISTEN:9001 SYSTEM:'cat /src/a.txt'

######Command for running :

    docker run -d --name myapp app1
    
######The content of a.txt:

    hello there how are you
        
######The Dockerfile for the second container:

        FROM ubuntu
        
        MAINTAINER Prashant Gupta
        
        RUN apt-get install -y curl
        
######Command for the second container:

    docker run --link myapp:server --rm -it --name myapp2 app2 curl server:9001
    
######Screencast:

![Image](https://github.com/prashantgupta24/Advanced-Docker/blob/master/Screencasts/hw4_1.gif)

2) **Ambassador pattern**: Implement the remote ambassador pattern to encapsulate access to a redis container by a container on a different host.

* Use Docker Compose to configure containers.
* Use two different VMs to isolate the docker hosts. VMs can be from vagrant, DO, etc.
* The client should just be performing a simple "set/get" request.
* In total, there should be 4 containers.

######Docker compose file for first VM:

        redis: 
          image: crosbymichael/redis
          container_name: redis
        
        ambassador:
          container_name: redis_ambassador
          image: svendowideit/ambassador
          links: 
            - redis:redis  
          ports: 
            - 6379:6379

######Docker compose file for second VM:

        redis_ambassador:
          image: svendowideit/ambassador
          container_name: redis_ambassador
          expose:
           - "6379"
          environment: 
            - REDIS_PORT_6379_TCP=tcp://192.168.1.52:6379
        
        redis_client: 
          image: anapsix/webdis
          container_name: redis_client
          links: 
            - redis_ambassador:redis
          ports:
           - 7379:7379

######Commands for running the redis client:

     curl 0.0.0.0/7379/ping ----> PONG
     curl 0.0.0.0/7379/SET/mykey/helloworld ----> OK
     curl 0.0.0.0/7379/GET/mykey ----> helloworld
     
######Screencast:

![Image](https://github.com/prashantgupta24/Advanced-Docker/blob/master/Screencasts/hw4_2.gif)

3) **Docker Deploy**: Extend the deployment workshop to run a docker deployment process.

* A commit will build a new docker image.
* Push to local registery.
* Deploy the dockerized [simple node.js App](https://github.com/CSC-DevOps/App) to blue or green slice.
* Add appropriate hook commands to pull from registery, stop, and restart containers.

######Post-commit hook. This builds the new image and pushes it to my local registry.

        #!/bin/bash
        
        docker rmi ncsu-app
        docker rmi localhost:5000/ncsu:latest
        docker build -t ncsu-app .
        docker tag ncsu-app localhost:5000/ncsu:latest
        docker push localhost:5000/ncsu:latest

######Adding the git remotes:

        git remote add blue "file:///home/prashant/Documents/Devops hw4/deploy/blue.git"
        git remote add green "file:///home/prashant/Documents/Devops hw4/deploy/green.git"

######Deploying the changes:

        git push green master
        git push blue master
        
######Post-commit hook for Blue slice:

        #!/bin/sh
        
        docker pull localhost:5000/ncsu:latest
        docker stop app2
        docker rm app2
        docker rmi localhost:5000/ncsu:current  
        docker tag localhost:5000/ncsu:latest localhost:5000/ncsu:current
        docker run -p 50102:8080 -d --name app2 localhost:5000/ncsu:latest

######Post-commit hook for Green slice:

        #!/bin/sh
        
        docker pull localhost:5000/ncsu:latest  
        docker stop app1  
        docker rm app1
        docker rmi localhost:5000/ncsu:current  
        docker tag localhost:5000/ncsu:latest localhost:5000/ncsu:current
        docker run -p 50101:8080 -d --name app1 localhost:5000/ncsu:latest 
        
######Screencast:

![Image](https://github.com/prashantgupta24/Advanced-Docker/blob/master/Screencasts/hw4_3.gif)
