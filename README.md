# Docker Kubernetes - The Practical Guide


## Core Building Block
Docker is a layer based architecture, bare this in mind when you order your commands on the Dockerfile, so you can leverage caching on building images.

### Useful commands
Use `docker --help`  to see the commands you can to run. You can choose to restart an existing container, for that do `docker ps` and then select the container id, and do `docker start <id>`. It is important to notice that with `start` the default mode is detached, while with `run` the default is attached mode (which doesn't allow us to execute more commands on the terminal). You can run a container in detach mode by passing the`-d` option to the command, or attach to a container with `docker attach <id>`.
Use `docker logs <id>` to inspect the container logs, or use `docker logs -f <id>` to follow the logs after you execute the command.
Use `docker rm <id>` to remove a stopped container, `docker rmi <id>` to remove an image, `docker images prune` to remove unused images (use `-a` option to remove all images). If you want to remove the container automatically when it is stopped use `--rm` option on docker run.Use `docker image inspect <id>` to get details of a docker image. 
To copy files into/out to a running container, we can do it with `docker cp <src> <dst>` where src is a local folder or a docker path in the form of <docker_name>:/<path> and dst is a path to copy to (the path will be created if it doesn't exists). 
You can name a container with the `--name` option to the run command. Also a tag can be given to an image, on build do: `docker build . -t <docker_name>:<tag>`.
Docker has build in commands to share images through Docker Hub or a private registry, for that create an account in docker hub and once your image is ready, do `docker push <repo_name>/<img_name>:<tag>` if you want to publish to docker hub, or use `docker push <host>:<port>/<repo_name>/<img_name>:<tag>` to push to a private registry. To pull images use `docker pull <image>`. If you want to retag an image so it is in sync with the remote to push it, use `docker tag <old_tag> <repo_name>/<img_name>`. If you want to push/pull from a private repo, make sure to login with `docker login` (use `docker logout` to close the session).


## Managing Data & Working with Volumes

### Volumes
Volumes are folders in the host machine, which are mapped to folders into docker containers to store non-volatile data. Containers can read/write data in this volumes. Use the `VOLUME ["path1", "path2",...]` indicating the paths within the container that should be persisted. This will add an anonymous volume that is managed by docker, which we can inspect using the `docker volume ls` instruction and only exists as long as the container that created it exists. To make it permanent, it is also possible to add a named volume using the option `-v <volume_name>:<conainer_path>` on container startup, and this volume won't be deleted after the container is shutdown (remove the `VOLUME` instruction if this option is used).

Bind mounts is an alternative to volumes, where we map a local folder into the container. Bind folders are created as well with the `-v <path_to_local_folder>:<path in the container>`. Enclose the argument passed in this command in quotes in case the path contains spaces. In case the path in the container that we are mapping the local folder into is not empty, the contents of this folder would be overwritten with the contents of the folder we are mapping, unless we merge the contents of both folders using anonymous volumes. Declare the path on your dockerfile using VOLUME or use `-v <volume_name>` (note we are not mapping or naming the volume, just declaring it). Docker uses a rule where the more specific (longer) path declared prevails, so we can map do something like `-v /tmp:/tmp` where the tmp folder in local is mapped into the container, but we can also declare a specific folder to be maintain within that path like this: `-v /tmp/path_to_keep`. To summarize:

    * Use `docker run -v <container_path` for anonymous volumes that won't be kept on container removed (can't share or save data).
    * use `docker run -v <name>:<container_path>` for named volumes that can be reused. Can't be created on the dockerfile, not tied to a container and survives shutdowns (can share or save data).
    * Use `docker run -v <local_path>:<container_path>` for bind mounts. Survives container removal, you need to remove from local to delete it. (can share or save data).

In case we are using bind mounts, we might want to make it read only to avoid the container overwritting some file system files. For that you use  `docker run -v <local_path>:<container_path>:ro`, the _ro_ at the end stands for read-only. You can still use an anonymous volume on that path and write on it (always passing it in the command line option, not on the docker file). i.e `docker run -v /tmp:/tmp:ro -v /tmp/app_data mycontainer` so we map _/tmp_ but because the container's _/tmp/app_data_ is more specific that _/tmp_, the configuration passed to create an anonymous volume overwrittes the config _ro_.
Volumes can also be created with the command line using `docker volumes create <volume_name>` and then use the volume on the run command. Check the volumes with `docker volumes inspect <volume_name>` and remove them with `docker volumes rm <volume_name>` (only if the volume is not in use).
The reason that we still use the COPY instruction having bind mounts available is that whatever you map to the container might not be transferrable (paths or files might not exists in other nodes), so you still want to copy those files in the images that would be deployed somewhere else, althought is still useful for development purposes. In the case you don't want to copy certain files into the image but you want to use a non specific copy instructions, you can exclude files and folders by using the  _.dockerignore_ file. just create this file at the same path where your Dockerfile is and put there the files and folders you want to exclude (i.e. Dockerfile or .git).

### Docker ARGuments and ENVironment variables
Docker supports image building arguments and runtime variables. To use environment variables (which can be used on your running app through the environment variables), we need to announce such variables in the Dockerfile using the instruction `ENV <ENV_NAME> <ENV_DEFAULT_VALUE>`, this variable can be subsequently used in the Dockerfile using _\$ENV_NAME_. To give a value to the container at runtime, use `--env <your_value>` or `-e <your_value>`. Optionally, you can pass a file containing the values with the `--env-file <path_to_env_file>`.
To be able to parameterize the builds, docker provide arguments which can be used in the form `ARG <ARG_NAME>=<DEFAULT_VALUE>` in the Dockerfile, and the use it subsequently with _$ARG_NAME_ (not in the CMD or ENTRYPOINT instructions as these are runtime commands). When you build the image, you can set other values by using `docker build -t <some_tag>:<version> --build-arg <ARG_NAME>=<OTHER_VALUE>` so the default value is overwritten by this one. Try to put ENV and ARG instructions at the bottom of the image, because the layers that goes after these instructions have to be rebuild for each different value, so the more layers you can keep unchanged, the better.


## Networking: (Cross-)Container Communication

There are several ocasions when we need our container to communicate with other services. The simplest is to sen request from inside the container to the world wide web which just works without any special setting. For communication between the running containers and the host machine, we need to replace the _localhost_ call with _host.docker.internal_ which is translated by docker with the IP of your localhost. Container to container you can run `docker container inspect <container_name>` and look for the IPAddress property in the resulting json, and use that one to communicate with the container, but this is not very usable, so you can also create container networks by passing `--network <network_name>` to the docker run command. After this, container that belongs to the same network, can communicate between them by using the names given or auto generated. For this to work, the network should have been previously created with `docker network create <network_name>`. When we launch a container that is used by other containers, you don't need to publish any port used by other containers, this is only required if we want to connect from our local machine or from outside docker.

Docker Networks actually support different kinds of "Drivers" which influence the behavior of the Network. The default driver is the "bridge" driver which provides the behavior described before. The driver can be set when a Network is created by adding the `--driver` option. Other options are:

    * host: For standalone containers, isolation between container and host system is removed (they share localhost as a network)
    * overlay: Multiple Docker daemons (i.e. Docker running on different machines) are able to connect with each other. Only works in "Swarm" mode (deprecated)
    * macvlan: You can set a custom MAC address to a container - this address can then be used for communication with that container
    * none: All networking is disabled
    * Third-party plugins: You can install third-party plugins which then may add all kinds of behaviors and functionalities


## Building Multi-Container Applications with Docker