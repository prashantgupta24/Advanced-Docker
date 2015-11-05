# Advanced-Docker
A repo containing the Ambassador pattern used in Dockers, also docker linking and building Docker images on Git commit

    docker run -d --name myapp app1
    docker run --link myapp:server --rm -it --name myapp2 app2 curl server:9001
