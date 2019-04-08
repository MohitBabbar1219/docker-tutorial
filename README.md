# Docker Tutorial

### Basics:
- `docker run <image-name>`<br/> Creates a container from the image and starts it in attached mode.
- The result of the above command can also be achieved in two separate steps:
    - `docker create <image-name>`<br/> Will return a hash (container id) that it just created.
    - `docker run <container-id>`<br/> Will run the container but in detached mode.
    - `docker run -a <container-id>`<br/> Will run the container in attached mode so that you can see the output of that container.
    - `docker logs <container-id>`<br/> Can also be used to see the output of any container. It comes in handy to see the logs of a detached container.
- Example:<br/> `docker run busybox echo hi there`<br/> Will run the command `echo hi there` on a container running busybox.<br/> But<br/> `docker run hello-world echo hi there`<br/> will not work as the busybox container did because both of them are running on different file systems where busybox has access to echo command but hello-world does'nt.
- There are two ways to stop/kill a container:
    - `docker stop <container-id>`<br/> Will issue a `SIGTERM` signal to the running process inside the container. Now the process can gracefully shutdown after doing the required clean-up.
    - `docker kill <container-id>`<br/> Will issue a `SIGKILL` signal to the running process inside the container. Now the process has to shut down immediately without doing any additional work.
- Time comes when multiple commands are to be executed on a container to work with it, for eg.:
    - `docker run redis`<br/> Will create and start a redis container. But if we run `redis-cli` on another terminal window of the host machine, it will give us an error because redis is running inside the container and does not expose it's commands outside. So we, somehow, need to run `redis-cli` inside the container.
    - `docker exec -it <container-id> <command>`<br/> Will run the command in the container. `-it` will run redis-cli in interactive mode which will expose the redis-cli repl to the host machine.
    - `docker exec <container-id> <command>`<br/> Will run the redis-cli but we will not get access to the redis-cli.
    - `docker exec <container-id> sh`<br/> Will get us full terminal access of the container.

### Creating our own images
- `Dockerfile` holds the configurations to define how our container should behave.
- Flow of a `Dockerfile`:
    1. Specify a base image
    2. Run some commands to install additional programs
    3. Specify a command to run on container start up
- `Dockerfile` == Being given a computer with no OS and being told to install Chrome.
- Refer to `redis-server/Dockerfile`. We create an `alpine` image with redis installed. A redis server is started on running this image.
- `docker build .`<br/> Will take the `Dockerfile` in the directory in which the command was executed, and generate an image out of it. But the built image can only be referenced by its hash. The `.` that we specify at the end of the command is the build context.
- `docker build -t <docker-id>/<project-name>:<version> .`<br/> Will build the image and tag it so that we don't need to call its hash to use it every time.  Technically, just the `<version>` is the tag.
- Another, not preferred, way of building images is by manually building them. For example, the redis server image can be built using the following steps:
    1. `docker run -it alpine bash`
    2. `apk add --update redis`
    3. `exit`
    4. `docker commit -c 'CMD ["redis-server"] <alpine-container-id>`<br/> Output is the id of our new image. This image will have same configurations as the one built with `Dockerfile`.

### Creating a simple node project
- We will try to build a simple express server and dockerize it. Refer to `simpleweb` directory for this web project.
- Now, the default `alpine` image that we have been using before will not just simply work. We will have to install `node` and `npm` on it or use a different image altogether which already has these things installed.
- We are going to use a `node` image for this project. Try to look for an image tagged `alpine`. `alpine`, in the docker world, is a term for an image that is as small and compact as possible.
- By default, the working directory will be the root directory of the container. We don't want to work in the root directory. So we can explicitly specify `WORKDIR /usr/app` so that all the commands will be executed in this directory. 
- Now that we've got the correct image, it still does'nt have access to our project files. We need to `COPY` the stuff we need the container to have from our host machine to the container.
- Next we need to install the project dependencies. `RUN` the command `npm install` to do that.
- The project is now set up in the container. Specify the final command which runs our node server: `CMD ["npm", "start"]`.
- `docker build -t <docker-id>/simpleweb .`<br/> The `Dockerfile` has everything we need for now, so we build it. Note that the `latest` tag is implicitly appended if not explicitly specified.
- `docker run -p 8080:8080 <docker-id>/simpleweb`<br/> The server will run on port `8080` of the container. If we go to `localhost:8080` on our host machine, it will not work as the port `8080` is not exposed to the host machine by default. To map the container's port to host's port, run the `docker run` command with `-p <port-number-on-host-machine>:<port-of-container>`. This will route incoming requests to `port on local host` to the `port inside the container`.  
- Great, you got your app running. But whenever you make a change in your project, however small it may be, the complete building process will happen all over again. The `npm install` command in itself can take ages if the project is significantly large. So, we need to break down the copying of files such that `package.json` is copied first and then `npm install` command is made to run. After the dependencies are installed, only then do we want to copy the project files. What this will achieve us is that the `package.json` file is gonna remain the same which means that dependencies are gonna remain the same, so, once built, the image will use the cached image till this step for all the other builds to come and will take a significantly lesser time to copy just the project files.

### Docker Compose with multiple docker containers
- In this section, we will build a simple multi-container app which will keep track of the number of visitors, refer `visits` directory. This project will have two components:
    1. Node server
    2. Redis server (in-memory data store)
- Dockerfile for this project will be similar to the previous project.
- Build the image and run the container. It will throw an error regarding no running redis server.
- `docker run redis`<br/> Will start the redis server, easy right?
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
    - `docker-compose up -d`<br/> Will start the containers in detached mode (in the background)
    - `docker-compose down`<br/> Will stop the running group of containers.
- We can also configure our project for failures such that if a service goes down, docker will know how to take care of it. So we make a route in our app (`/exit_server`) (to simulate an error) which, when visited, will shutdown the server with status code 1. Status code of 0 states that everything went as expected and the server was meant to exit, but the exit status code of anything beside 0 (i.e. 1, 2, 3, ...) states that server exited because something went wrong.
- Docker provides `restart policies` to control whether your container should restart automatically when they exit, or when docker restarts. To configure the restart policies, any of the following can be used:
    - `no` - Do not automatically restart the container.
    - `on-falure` - Restart the container if it exits due to an error, which manifests as a non-zero exit code.
    - `always` - Always restart the container if it stops. If it is manually stopped, it is restarted only when Docker daemon restarts or the container itself is manually restarted.
    - `unless-stopped` - Similar to always, except that when the container is stopped (manually or otherwise), it is not restarted even after Docker daemon restarts.
- The above restart policies can be used with `docker run` command with `--restart` flag or can be specified in the `docker-compose.yml` file, we're going to do the latter.

### Using Docker in production grade environment
- Workflow to publish an app is a closed loop that starts with development, then testing and then deployment and at some later point of time, development again.
- Github repository will hold all the code that we write and eventually deploy to some outside service. Our github repo is going to have two different type of branches:
    1. Feature branch - A development branch in which we are going to add the code or change it in order to update our app. Once we've pushed the changes to the feature branch, we'll raise a pull request and merge them over to the master branch. Once we're done with the pull request, two things are going to occur:
        1. A workflow will take our application and push it to a service called Travis CI, it will run a set of tests that we write.
        2. Then, Travis CI is going to push our project to our hosting service, say AWS.
    2. Master branch - Will represent the clean, working copy of our codebase. Any changes made to the master branch will eventually, automatically, be deployed out to our hosting provider (such as AWS, GCP, etc.).
- Where does Docker fit in this workflow? Docker is not a requirement for using this workflow, but it is going to make a lot of these tasks a lot easier. Keep in mind that Docker is not our primary focus here.
- We will be working with a react app in this chapter. Dockerfile is going to be similar to the Dockerfile in previous projects, there's just one subtle difference, we're going to have to Dockerfiles in this project, namely, `DockerfileDev` (for development environment) and `Dockerfile` (for production environment). To build an image with a custom named Docker file, use `-f` tag to specify the name, command:<br/> `docker build -f <dockerfile-name> .`
- Do you remember that we used to `COPY` all the code and build it in the container to run our projects? This won't work anymore in our development environment as it will require us to build the image again and again whenever we make a change in our app's code, however small it may be. We need to abandon this approach of copying code to container and building images.
- Luckily, Docker gives us a feature called `volumes`. We can now provide a reference to the directory which holds all the code, to the container such that the container can access that folder on the host machine as if it is its own.
- `docker run -p <host-port>:<container-port> -v /usr/app/node_modules -v $(pwd):/usr/app <image-id>`<br/> We'll break it down:
    1. `-v $(pwd):/usr/app` Will map the current working directory that this command is executed in, to `/usr/app` of the container. This means that the content of current working directory of host machine will reflect in `/usr/app` directory of the container
    2. `-v /usr/app/node_modules` Will tell docker that don't try to map this folder from the host machine. Notice that there is no `:` present in this tag, this allows us to skip it's mapping. This allows us to not keep the `node_modules` folder on host machine.
- The only thing not elegant about this solution is the unusually long and ugly `docker run` command. Even though we have only a single image this time, we can still use `docker-compose` (refer `frontend/docker-compose.yml`) and make it a workaround for the not-so-elegant `docker run` command.
- We created a service called `react-app` in our `docker-compose.yml` file to run our app in a development environment. Now, we can either create another service which will solely be responsible for running our app's tests or we can run<br/> `docker exec -it <container-id> bash`<br/> and then run<br/>`npm run test`<br/> inside the container's shell to get the interactive testing window.
- When we build our app for the production environment, the development server that we use in the development environment goes away and all that is left is a js file and other static html and css files. To serve these files in a production environment we'll need another server. We are going to use the `nginx` server as a production server.
- This will be interesting because now we need `node:alpine` image to build our project and `nginx` image to run it in production environment. Looks like we are in a situation where two base images will be nice to have.
- What we are going to do is we are going to build a `Dockerfile` (refer `frontend/Dockerfile`) which has a multi-step build process. It will have two different blocks of configuration:
    1. Build phase
        1. `COPY` the `package.json` file.
        2. Install dependencies
        3. `RUN` npm run build
    2. Run phase
        1. Use `nginx`
        2. Copy over the result of Build phase
        3. Start `nginx`, this does'nt need an explicit command.
- We can now build and run the image with `Dockerfile` and this app will be production ready as `nginx` is a production ready server.

### CI and CD with AWS
- We are going to integrate out github repo (refer to [react-docker](https://github.com/MohitBabbar1219/docker-react)) with Travis CI which will test our app and deploy it to AWS.
- Sign up for Travis CI and link your repo to it.
- Now, whenever you push your changes to your github repo, Travis CI will automatically start the build process, provided you have `.travis.yml` file (refer `.travis.yml` in the repo) present in your repo.
- Flow of `.travis.yml` file:
    1. `sudo: required`<br/> Will make the process happen with su permissions
    2. `services`<br/> Will specify the service that we are going to need which will facilitate project building
    3. `before_install`<br/> Will run the prerequisite commands
    4. `script`<br/> Will hold the main commands to test or build the project. If any of these fails, the pipeline will fail.
    5. `deploy`<br/> Will hold the configuration and commands to deploy the project. 
- Moving on to AWS, Elastic beanstalk is the easiest way to get started with production docker instances. It provides a load balancer out of the box and makes deployment of docker containers a breeze.
- `deploy` section in the `.travis.yml` file will take care of the deployment, just remember to `EXPOSE 80` port in your Dockerfile as this is the default port that our sever will be listening on.

### Building a multi-container app
- For this section, we are going to over-complicate a simple fibonacci sequence calculator.
- Components of our app (All the source code can be found in `complex` directory):
    1. `nginx` server
    2. `react` server
    3. `express` server
    4. `redis` in-memory data store
    6. `worker` watcher that updates redis store when something new happens
    5. `postgres` database
- Development Dockerfile(s) for our setup (refer to `<service-name>/DockerfileDev`):
    1. `client` is a react app. We will be using the same Dockerfile flow as the previous react app.
    2. `server` and `worker` are node apps. Dockerfile(s) for these apps will also be similar to the react app Dockerfile.
- `docker-compose.yml` file for this setup is going to be interesting. Flow of this file (refer to `complex/docker-compose.yml`):
    1. `version`
    2. `services`:
        - `postgres`: What image to use?
        - `redis`: What image to use?
        - `server`: Specify `build`, `volumes` and `environment` variables.
        - `client`: Specify `build` and `volumes`.
        - `worker`: Specify `build` and `volumes`.
        - `nginx`: Specify `build`, `restart` and `ports`.
- Environment variables in `docker-compose.yml`:
    1. `VAR_NAME=<VALUE>` Sets a variable in the container at runtime.
    2. `VAR_NAME` Sets a variable in the container at runtime but the value is taken from your computer.
    3. `.env` File will extract them from the file specified.

### CI workflow for a multi-container setup
- Workflow of a multi-container setup:
    1. Push code to github
    2. Travis automatically pulls the repo, builds a test image and tests the code
    3. Travis builds the prod images
    4. Travis pushes built prod images to docker hub
    5. Travis pushes the project to AWS EB
    6. EB pulls the images from docker hub and deploys
- Refer `.travis.yml` for the CI setup for this project. Also note that we are using the production `Dockerfile`s for building the images now.

### Deploying multiple containers to AWS
- Deploying single container was straight forward. To deploy a multi-container project, we have to put the configurations of all the containers in one single file for AWS to know, we'll be calling that file `Dockerrun.aws.json`.
- This `Dockerrun.aws.json` will tell elastic beanstalk where to pull the images from, what resources to allocate to each container, how to setup port mappings and other vital information. Elastic Beanstalk also works with ECS behind the scenes, so we need to define task definitions (refer docs for task definitions) in the file.
- Format of the `Dockerrun.aws.json` file:
    1. `AWSEBDockerrunVersion`
    2. `containerDefinitions`:
        1. `name`: Name of the service, eg. `client`
        2. `image`: Our image on docker hub
        3. `hostname`: Address at which all the other services will communicate with this service. Eg. `client`
        4. `essential`: Boolean which decides if the current service is as essential as terminating the other services if this one goes down. Eg. our `nginx` server could be marked essential.
- We need to create a new application and environment, appropriate to our use case, on elastic beanstalk for our app.
- AWS provides two other different services for hosting DBs and in-memory data stores, namely AWS relational database service and AWS elastic cache, respectively.
- Since, RDS and EC are different services, we need to define a security group to allow them to communicate to each other.
