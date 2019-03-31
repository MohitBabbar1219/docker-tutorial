# Docker Tutorial

### Basics:
- `docker run <image-name>` creates a container from the image and starts it in attached mode.
- The result of the above command can also be achieved in two separate steps:
    - `docker create <image-name>` Will return a hash (container id) that it just created.
    - `docker run <container-id>` Will run the container but in detached mode.
    - `docker run -a <container-id>` Will run the container in attached mode so that you can see the output of that container.
    - `docker logs <container-id>` Can also be used to see the output of any container. It comes in handy to see the logs of a detached container.
- Example: `docker run busybox echo hi there` Will run the command `echo hi there` on a container running busybox. But `docker run hello-world echo hi there` will not work as busybox container did because both of them are running on different file systems where busybox has access to echo command but hello-world does'nt.
- There are two ways to stop/kill a container:
    - `docker stop <container-id>` Will issue a `SIGTERM` signal to the running process inside the container. Now the process can gracefully shutdown after doing the required clean-up.
    - `docker kill <container-id>` Will issue a `SIGKILL` signal to the running process inside the container. Now the process has to shut down immediately without doing any additional work.
- Time comes when multiple commands are to be executed on a container to work with it, for eg.:
    - `docker run redis` Will create and start a redis container. But if we run `redis-cli` on another terminal window of the host machine, it will give us an error because redis is running inside the container and does not expose it's commands outside. So we, somehow, need to run `redis-cli` inside the container.
    - `docker exec -it <container-id> <command>` Will run the command in the container. `-it` will run redis-cli in interactive mode which will expose the redis-cli repl to the host machine.
    - `docker exec <container-id> <command>` Will run the redis-cli but we will not get access to the redis-cli.
    - `docker exec <container-id> sh` Will get us full terminal access of the container.

### Creating our own images
- `Dockerfile` holds the configurations to define how our container should behave.
- Flow of a `Dockerfile`:
    1. Specify a base image
    2. Run some commands to install additional programs
    3. Specify a command to run on container start up
- `Dockerfile` == Being given a computer with no OS and being told to install Chrome.
- Refer to `redis-server/Dockerfile`. We create an `alpine` image with redis installed. A redis server is started on running this image.
- `docker build .` Will take the `Dockerfile` in the directory that the command was run, and generate an image out of it. But the built image can only be referenced by its hash.
- `docker build -t <docker-id>/<project-name>:<version> .` Will build the image and tag it so that we don't need to call its hash to use it every time.  Technically, just the `<version>` is the tag.
- Another, not preferred, way of building images is by manually building them. For example, the redis server image can be built using the following steps:
    1. `docker run -it alpine bash`
    2. `apk add --update redis`
    3. `exit`
    4. `docker commit -c 'CMD ["redis-server"] <alpine-container-id>` Output is the id of our new image. This image will have same configurations as the one built with `Dockerfile`.

### Creating a simple node project
- We will try to build a simple express server and dockerize it. Refer to `simpleweb` directory for this web project.
- Now, the default `alpine` image that we have been using before will not just simply work. We will have to install `node` and `npm` on it or use a different image altogether which already has these things installed.
- We are going to use a `node` image for this project. Try to look for an image tagged `alpine`. `alpine`, in the docker world, is a term for an image that is as small and compact as possible.
- By default, the working directory will be the root directory of the container. We don't want to work in the root directory. So we can explicitly specify `WORKDIR /usr/app` so that all the commands will be executed in this directory. 
- Now that we've got the correct image, it still does'nt have access to our project files. We need to `COPY` the stuff we need the container to have from our host machine to the container.
- Next we need to install the project dependencies. `RUN` the command `npm install` to do that.
- The project is now set up in the container. Specify the final command which runs our node server: `CMD ["npm", "start"]`.
- `docker build -t <docker-id>/simpleweb .` The `Dockerfile` has everything we need for now, so we build it. Note that the `latest` tag is implicitly appended if not explicitly specified.
- `docker run -p 8080:8080 <docker-id>/simpleweb` The server will run on port `8080` of the container. If we go to `localhost:8080` on our host machine, it will not work as the port `8080` is not exposed to the host machine by default. To map the container's port to host's port, run the `docker run` command with `-p <port-number-on-host-machine>:<port-of-container>`. This will route incoming requests to `port on local host` to the `port inside the container`.  
- Great, you got your app running. But whenever you make a change in your project, however small it may be, the complete building process will happen all over again. The `npm install` command in itself can take ages if the project is significantly large. So, we need to break down the copying of files such that `package.json` is copied first and then `npm install` command is made to run. After the dependencies are installed, only then do we want to copy the project files. What this will achieve us is that the `package.json` file is gonna remain the same which means that dependencies are gonna remain the same, so, once built, the image will use the cached image till this step for all the other builds to come and will take a significantly lesser time to copy just the project files.

### Docker Compose with multiple docker containers
- In this section, we will build a simple multi-container app which will keep track of the number of visitors, refer `visits` directory. This project will have two components:
    1. Node server
    2. Redis server (in-memory data store)
- Dockerfile for this project will be similar to the previous project.
- Build the image and run the container. It will throw an error regarding no running redis server.
- `docker run redis` Will start the redis server, easy right?
- If you run your node server again, it will still complain about the redis server not being up. This is because the express server and the redis server are two separate and isolated containers which have no medium of communication with each other. So, we need to set up a networking infrastructure between the two.
- There are two ways to set up this networking functionality:
    1. Use Docker CLI's networking features - but this is not straightforward and will require execution of a lot of different commands every time you start up your multi-container application.
    2. Use Docker Compose - Used to start up multiple docker containers at the same time and automate some of the long arguments we were passing to the `docker run` command.
- `docker-compose.yml` is the YAML file used which will contain all the configurations for our application's services (redis server and node server). Then, with a single command, we create and start all the services with the specified configurations.
- Flow of the `docker-compose.yml` file for our project (Refer `visits/docker-compose.yml`):
    - The containers that we want created:
        1. Redis server
            - Make it using the redis image
        2. Node app
            - Make it using the Dockerfile
            - Map port 8081 to 8081
- Using `docker-compose` to initialize the services will automatically run both containers on the same network and they can have free access to communicate amongst themselves.
- When configuring `redisClient` in the node project, we specify host as `redis-server`, same as the docker service name for redis server. Now, the `redisClient` will try to make a connection request to address `redis-server` (ideally it should have been an address like 'https://my-redis-server.com') assuming that it is a meaningful url, when the connection request is made, docker is going to know that the request is meant for another container named `redis-server` and will redirect it to .
- Now, multiple containers can be started and managed with simple `docker-compose` commands. Run these commands in the directory that holds `docker-compose.yml` file:
    - `docker-compose up -d` Will start the containers in detached mode (in the background)
    - `docker-compose down` Will stop the running group of containers.
- We can also configure our project for failures such that if a service goes down, docker will know how to take care of it. So we make a route in our app (`/exit_server`) (to simulate an error) which, when visited, will shutdown the server with status code 1. Status code of 0 states that everything went as expected and the server was meant to exit, but the exit status code of anything beside 0 (i.e. 1, 2, 3, ...) states that server exited because something went wrong.
- Docker provides `restart policies` to control whether your container should restart automatically when they exit, or when docker restarts. To configure the restart policies, any of the following can be used:
    - `no` - Do not automatically restart the container.
    - `on-falure` - Restart the container if it exits due to an error, which manifests as a non-zero exit code.
    - `always` - Always restart the container if it stops. If it is manually stopped, it is restarted only when Docker daemon restarts or the container itself is manually restarted.
    - `unless-stopped` - Similar to always, except that when the container is stopped (manually or otherwise), it is not restarted even after Docker daemon restarts.
- The above restart policies can be used with `docker run` command with `--restart` flag or specified in the `docker-compose.yml` file, we're going to do the latter. 
