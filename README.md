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
    3. Specify a command to run on container startup
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
