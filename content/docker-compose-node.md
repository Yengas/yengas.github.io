+++
title = "Using Docker and Docker-Compose to setup a local working environment, ready to be scaled."
date = "2017-01-31T03:30:00+03:00"
Categories = ["DevOps"]
Tags = ["docker", "kubernetes", "node", "nodemon", "forever", "local" ]
+++

Let's talk about running your Node.js applications, in a way that makes it easier to migrate to continuous integration and deployment pipelines later on the road. There are too many things that you can do in DevOps layer that will make your code more readable and easier to maintain. For example:
 
 - You can assume that every **configuration** is retrieved through a single file, then make a spec in your Kubernetes cluster to dynamically create this file through [ConfigMaps](https://kubernetes.io/docs/user-guide/configmap/), which would solve your problem of moving/managing configuration files on multiple servers. You can also mount these [ConfigMaps](https://kubernetes.io/docs/user-guide/configmap/) as environment variables.
 - You can make your **service discovery** by making the hostnames/ports of the services you're using variable. This will give you the flexibility of either using an intermediary DNS server(e.g. [kube-dns](https://kubernetes.io/docs/admin/dns/)), or entering static values(e.g. `localhost` or docker service name).
- You can **log** everything to `stdout` in your code. Then use a collector like [fluentd](http://www.fluentd.org/) to forward your logs to a centralized logging solution to analyze and report on.
- You can leave out anything related to simple cpu/memory **monitoring**. Then use something like [heapster](https://github.com/kubernetes/heapster) to retrieve these data through docker.
- You can leave out any **rate limiting** to be handled before any request reaches to your services by putting up a reverse proxy in front of your services.
- You can forget about **mapping domains/paths** to backend services and use dynamic reverse-proxy configurations(e.g. [Kubernetes ingress controller](https://github.com/kubernetes/contrib/tree/master/ingress/controllers)) to handle it in your operations layer.
- You can assume that a specific folder is shared between multiple instances of your processes, then use [NFS volume mounts](https://github.com/kubernetes/kubernetes/tree/master/examples/volumes/nfs) to have a truly shared filesystem between multiple nodes in your cluster. 

These are all helpful... but maybe hard for you to setup all at once, or you just may not have the resources. This doesn't mean you should code your services like you don't have these options. You can mostly have the juicy parts of these features just by using Docker and some clever structuring/configuration. Below i will explain how to achieve a working environment that will make it easy for you to push your services into Kubernetes.

## Configuration
Have a single configuration file that retrieves your environment specific configurations through environment variables, and holds the other kind of information(configuration you would like to have in your source control) as hardcoded.

This will make it possible for you to plug-in different kinds of environment files on different environments. You can achieve this by using `env_file` in `docker-compose` or `ConfigMap` in Kubernetes. You also have the chance to dynamically create this file in your docker entrypoint and override it with a configuration retrieved from a database or somewhere else.

- Configurations that changes from host to host or environment to environment should be retrieved as environment variables or files.
- Configurations that change the business logic related behaviour of the application should be stored within scm(git, svn).

### Single Configuration File
```javascript
// config.js
module.exports = {
  // Good old way of getting configurations.
  database: { password: process.env.DATABASE_PORT || 3306 },
  // Multi-level objects can be stored as .json files and then serialized into ENV variables.
  session: JSON.parse(process.env.SESSION_OPTIONS || '{}'),
  // Magic string should be in our scm
  magic: { string: 'Hello, World!' }
};
```
After having a configuration file like this. You can create `.env.example` to hold the default values, and use specific configurations for each environments (as: `.env.{ENVIRONMENT}`). These files can be loaded with docker-compose to your container's `process.env`. You can also create Kubernetes spec to mount Environment Variables from `ConfigMaps`.

### Docker Envfile (Example)
```yaml
# docker-compose.yaml
version: '2'
services:
    application:
      image: xxx:latest
      env_file: ./.env.development
```

## Shared Files
You can use docker named volumes for folders you would like to share between multiple instances of your services, or to store persistent information between container re-creates.

There is also the ability to create named volumes backed by AWS storage or NFS. You're almost as flexible as you would have been with Kubernetes and it's much more easier.

### Named Volumes (Example)
Example:
```yaml
# docker-compose.yaml
version: '2'
volumes:
  database-storage: {}
services:
    database:
      image: mariadb:10.1
      volumes:
        - database-storage:/var/lib/mysql
```

## Service Discovery
You should make all of your remote dependencies host names and ports configurable. Use the suggested single configuratiion file strategy to hold your host name and port information. You can use static ip addresses like `localhost` or public ips in one environment configuration, and use custom hostnames like `database` in another to let it resolve through a DNS.

Docker supports DNS resolving of containers that run in the same network. You can easily run 2 different services called `database` and `application` and both will see each other in their given name, thanks to the DNS service supplied by Docker. You can also add extra hosts with static ip resolutions using `extra_hosts`.

### Docker Networks (Example)
```yaml
# docker-compose.yaml
version: "2"
networks:
  redisnetwork: { driver: bridge }
services:
  application:
    image: xxx:latest
    # Extra host is defined here.
    # This service can resolve both database and google hostnames.
    extra_hosts: &default_hosts
      - "google:172.217.17.174"
  database:
    build: mariadb:10.1
    # This service can resolve both application and google hostnames.
    extra_hosts: *default_hosts
```

Only problem with this setup is when you would like to spin up multiple instances of a given container. You can spin up a reverse proxy container to load balance the recv requests to itself and forward them to other services defined in the docker-compose file. With Kubernetes, you can use a `Service`.

## Hot Reloading
Most of the continuous integration pipelines are built to output a single docker image that could be run as if were a single binary. This means adding your source code into the image, installing all dependencies and making the main file of your project the entrypoint. However a Dockerfile designed for this process is not very useful when working locally. Since the source code is added to the image in the `docker build` process, changes you make on your code after the build phase will not change how the containers created by the previously built image will behave.

This is why i have 2 different Dockerfiles, one for building `binary` images and one for development. The trick with development Dockerfile is that we add our source code into the container, not into the image. This means mounting the source directory from the Host to the Docker container. Things you need to take into consideration here is as follows;

### Docker Container Volume Mounts
Docker containers can mount directories from where the docker engine runs, not from the machine you run the `docker` command from. Meaning if you're using Virtualbox or a remote server as your Docker Host, this virtual/remote machine should have access to your source code. Since we're talking about local development. You're probably running docker-engine and docker cli in the same machine, or you have your docker-engine in a virtual box.

In case of a Virtualbox, make sure your source code's root folder is shared with the guest os. This is automatically configured for Windows Toolbox. Toolbox mounts `C:\Users` into `/c/users` of the docker-engine guest os. However the docker-compose that comes with the Toolbox can't mount from `/c/users`. You will have a weird error about how the volume mount includes invalid characters, and you should use absolute paths. In this case you should set `COMPOSE_CONVERT_WINDOWS_PATHS` environment variable to `1` in the terminal you run docker-compose.

In other scenarios like running docker-compose under Hyper-V or native Linux, there shouldn't be any problems.

### Dependency Installation
You should never mount your `node_modules` into a container. There are packages which require libraries that are specifically built for your operating system. In these cases, mounting an incompatible library will cause your application to not work.

This is why you should run `npm install` inside the image and only mount the source code needed to run your service. This may require you to `build-essentials` into your image. If you're using an image that doesn't have it like `alpine-node`.

### Utilities
You will need utilities to watch for code changes and restart your application. Easiest to setup utilities and most frequently used ones are `forever` and `nodemon`. However keep in mind that `nodemon` needs to exit when your application crashes, so forever will pick up that your application has been terminated, and restart it. This can be done using the `--exitcrash` flag of `nodemon`.

And another thing to keep in mind is that `nodemon` uses `inotify` to watch for file changes. This may not work in case you're using Hyper-V or Virtualbox. You can fix this by using polling.

### Development Dockerfile (Example)
```dockerfile
# Dockerfile
FROM alpine-node:6.9.1

RUN mkdir /application

RUN npm install -g nodemon
RUN npm install -g forever

WORKDIR /application

ADD ./package.json package.json
RUN npm install

VOLUME /application/src

EXPOSE 80

CMD forever --spinSleepTime 10000 --minUptime 5000 -c 'nodemon --exitcrash -L --watch /application/src' /application/src/index.js
```

This will create an image that will have all the dependencies installed for a given nodejs service. You can then mount the source code from your host to a container which was created by this image, and you will have a hotreloading nodejs microservice that restart everytime you make a change.

An accompanying docker-compose file should be like this:
```yaml
version: '2'
services:
    application:
      build: ./application
      volumes:
       - ./application/src:/application/src 
```

## Conclusion
We talked about what we can gain using DevOps. Instead of manually scripting solutions into our business logic, we can use containers and container runtimes to make it easy to reason about general problems of microservices. We showed that we didn't need any complex setups or configurations to start coding our services in a way that makes them more readable, and easier to migrate to container runtimes.
 
With the advance of technologies such as Docker and Kubernetes, your services should only include code about your business logic. Not about your infrastructure.

### A local working environment (Example)
Below is an example of backend application using Mariadb database with prepopulated sql data.
```yaml
# docker-compose.yaml
version: '2'
# Network for creating isolated container groups
networks:
  # Server group that contains database and application.
  servernetwork: { driver: bridge }
# Named volumes for tidy storage on docker host.
volumes:
  # Database storage volume.
  database-storage: {}
services:
  database:
    image: mariadb:10.1
    ports:
      # Expose the port to localhost so you can connect with a database client.
      - "3306:3306"
    volumes:
      # Make the data persistent in a named volume.
      - database-storage:/var/lib/mysql
      # Add startup state for the Mariadb container.
      - ./database/backup:/docker-entrypoint-initdb.d
    networks:
      # Add to the server network
      - servernetwork
    environment:
      # make Mariadb run with empty pass
      - MYSQL_ALLOW_EMPTY_PASSWORD=true
  application:
    build:
      # Change the build context to the development file.
      # Its always nice to have the 'binary` building dockerfile as the default.
      context: ./application
      dockerfile: Dockerfile-development
    volumes:
      # Link the host directory to container.
      - ./application/src:/application/src
    env_file:
      # Get the configuration from the dotenv file.
      - ./application/.env.development
    ports:
      # Expose the port.
      - "80:80"
    networks:
      # Add the port.
      - servernetwork
    depends_on:
      # make sure the database starts up with application.
      # doesn't guarantee that the database will be ready before backend works.
      - database
```


