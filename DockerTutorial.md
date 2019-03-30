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
