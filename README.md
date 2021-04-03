# Docker Kubernetes - The Practical Guide

## Core Building Block

Docker is a layer based architecture, bare this in mind when you order your commands on the Dockerfile, so you can leverage
caching on building images.

### Useful commands

Use `docker --help`  to see the commands you can to run. You can choose to restart an existing container, for that
do `docker ps` and then select the container id, and do `docker start <id>`. It is important to notice that with `start` the
default mode is detached, while with `run` the default is attached mode (which doesn't allow us to execute more commands on
the terminal). You can run a container in detach mode by passing the`-d` option to the command, or attach to a container
with `docker attach <id>`. Use `docker logs <id>` to inspect the container logs, or use `docker logs -f <id>` to follow the
logs after you execute the command. Use `docker rm <id>` to remove a stopped container, `docker rmi <id>` to remove an
image, `docker images prune` to remove unused images (use `-a` option to remove all images). If you want to remove the
container automatically when it is stopped use `--rm` option on docker run.Use `docker image inspect <id>` to get details of
a docker image. To copy files into/out to a running container, we can do it with `docker cp <src> <dst>` where src is a local
folder or a docker path in the form of <docker_name>:/<path> and dst is a path to copy to (the path will be created if it
doesn't exists). You can name a container with the `--name` option to the run command. Also a tag can be given to an image,
on build do: `docker build . -t <docker_name>:<tag>`. Docker has build in commands to share images through Docker Hub or a
private registry, for that create an account in docker hub and once your image is ready,
do `docker push <repo_name>/<img_name>:<tag>` if you want to publish to docker hub, or
use `docker push <host>:<port>/<repo_name>/<img_name>:<tag>` to push to a private registry. To pull images
use `docker pull <image>`. If you want to retag an image so it is in sync with the remote to push it,
use `docker tag <old_tag> <repo_name>/<img_name>`. If you want to push/pull from a private repo, make sure to login
with `docker login` (use `docker logout` to close the session).

## Managing Data & Working with Volumes

### Volumes

Volumes are folders in the host machine, which are mapped to folders into docker containers to store non-volatile data.
Containers can read/write data in this volumes. Use the `VOLUME ["path1", "path2",...]` indicating the paths within the
container that should be persisted. This will add an anonymous volume that is managed by docker, which we can inspect using
the `docker volume ls` instruction and only exists as long as the container that created it exists. To make it permanent, it
is also possible to add a named volume using the option `-v <volume_name>:<conainer_path>` on container startup, and this
volume won't be deleted after the container is shutdown (remove the `VOLUME` instruction if this option is used).

Bind mounts is an alternative to volumes, where we map a local folder into the container. Bind folders are created as well
with the `-v <path_to_local_folder>:<path in the container>`. Enclose the argument passed in this command in quotes in case
the path contains spaces. In case the path in the container that we are mapping the local folder into is not empty, the
contents of this folder would be overwritten with the contents of the folder we are mapping, unless we merge the contents of
both folders using anonymous volumes. Declare the path on your dockerfile using VOLUME or use `-v <volume_name>` (note we are
not mapping or naming the volume, just declaring it). Docker uses a rule where the more specific (longer) path declared
prevails, so we can map do something like `-v /tmp:/tmp` where the tmp folder in local is mapped into the container, but we
can also declare a specific folder to be maintain within that path like this: `-v /tmp/path_to_keep`. To summarize:

    * Use `docker run -v <container_path` for anonymous volumes that won't be kept on container removed (can't share or save data).
    * use `docker run -v <name>:<container_path>` for named volumes that can be reused. Can't be created on the dockerfile, not tied to a container and survives shutdowns (can share or save data).
    * Use `docker run -v <local_path>:<container_path>` for bind mounts. Survives container removal, you need to remove from local to delete it. (can share or save data).

In case we are using bind mounts, we might want to make it read only to avoid the container overwritting some file system
files. For that you use  `docker run -v <local_path>:<container_path>:ro`, the _ro_ at the end stands for read-only. You can
still use an anonymous volume on that path and write on it (always passing it in the command line option, not on the docker
file). i.e `docker run -v /tmp:/tmp:ro -v /tmp/app_data mycontainer` so we map _/tmp_ but because the container's _
/tmp/app_data_ is more specific that _/tmp_, the configuration passed to create an anonymous volume overwrittes the config _
ro_. Volumes can also be created with the command line using `docker volumes create <volume_name>` and then use the volume on
the run command. Check the volumes with `docker volumes inspect <volume_name>` and remove them
with `docker volumes rm <volume_name>` (only if the volume is not in use). The reason that we still use the COPY instruction
having bind mounts available is that whatever you map to the container might not be transferrable (paths or files might not
exists in other nodes), so you still want to copy those files in the images that would be deployed somewhere else, althought
is still useful for development purposes. In the case you don't want to copy certain files into the image but you want to use
a non specific copy instructions, you can exclude files and folders by using the  _.dockerignore_ file. just create this file
at the same path where your Dockerfile is and put there the files and folders you want to exclude (i.e. Dockerfile or .git).

### Docker ARGuments and ENVironment variables

Docker supports image building arguments and runtime variables. To use environment variables (which can be used on your
running app through the environment variables), we need to announce such variables in the Dockerfile using the
instruction `ENV <ENV_NAME> <ENV_DEFAULT_VALUE>`, this variable can be subsequently used in the Dockerfile using _\$ENV_NAME_
. To give a value to the container at runtime, use `--env <your_value>` or `-e <your_value>`. Optionally, you can pass a file
containing the values with the `--env-file <path_to_env_file>`. To be able to parameterize the builds, docker provide
arguments which can be used in the form `ARG <ARG_NAME>=<DEFAULT_VALUE>` in the Dockerfile, and the use it subsequently
with _$ARG_NAME_ (not in the CMD or ENTRYPOINT instructions as these are runtime commands). When you build the image, you can
set other values by using `docker build -t <some_tag>:<version> --build-arg <ARG_NAME>=<OTHER_VALUE>` so the default value is
overwritten by this one. Try to put ENV and ARG instructions at the bottom of the image, because the layers that goes after
these instructions have to be rebuild for each different value, so the more layers you can keep unchanged, the better.

## Networking: (Cross-)Container Communication

There are several ocasions when we need our container to communicate with other services. The simplest is to sen request from
inside the container to the world wide web which just works without any special setting. For communication between the
running containers and the host machine, we need to replace the _localhost_ call with _host.docker.internal_ which is
translated by docker with the IP of your localhost. Container to container you can
run `docker container inspect <container_name>` and look for the IPAddress property in the resulting json, and use that one
to communicate with the container, but this is not very usable, so you can also create container networks by
passing `--network <network_name>` to the docker run command. After this, container that belongs to the same network, can
communicate between them by using the names given or auto generated. For this to work, the network should have been
previously created with `docker network create <network_name>`. When we launch a container that is used by other containers,
you don't need to publish any port used by other containers, this is only required if we want to connect from our local
machine or from outside docker.

Docker Networks actually support different kinds of "Drivers" which influence the behavior of the Network. The default driver
is the "bridge" driver which provides the behavior described before. The driver can be set when a Network is created by
adding the `--driver` option. Other options are:

    * host: For standalone containers, isolation between container and host system is removed (they share localhost as a network)
    * overlay: Multiple Docker daemons (i.e. Docker running on different machines) are able to connect with each other. Only works in "Swarm" mode (deprecated)
    * macvlan: You can set a custom MAC address to a container - this address can then be used for communication with that container
    * none: All networking is disabled
    * Third-party plugins: You can install third-party plugins which then may add all kinds of behaviors and functionalities

## Building Multi-Container Applications with Docker

On Example 5 you need the following commands:

``` shell
# Create the network
docker network create goals-net

# build backend
cd backend
docker build -t goals-node .

# build frontend
cd ../frontend
docker build -t goals-react .

# Spin up mongo db
docker run --name mongodb --rm -d -v data:/data/db --network goals-net -e MONGO_INITDB_ROOT_USERNAME=max -e MONGO_INITDB_ROOT_PASSWORD=secret mongo

# Spin up backend
docker run -name goals-backend --rm -d -v /Users/jorge/Documents/git-repos/docker-kubernetes-practical-guide/Example5/backend:/app -v logs:/app/logs -v /app/node_modules -p 80:80 --network goals-net goals-node

# Spin up frontend
docker run -v /Users/jorge/Documents/git-repos/docker-kubernetes-practical-guide/Example5/frontend/src:/app/src -name goals-react --rm -d -it -p 3000:3000 goals-react
```

## Building Multi-Container Applications with DockerDocker Compose: Elegant Multi-Container Orchestration

Docker Compose is a tool that allows you to replace docker build and run commands with just one configuration file and then a
set of orchestration commands to start all those services, all these containers at once and build all necessary images, if it
should be required. Several sections can be added to a docker compose file, but the most important is the services section,
which specifies the containers configuration, within this section you can define ports, environment variables, volumes,
networks... To work with docker compose you need to define a _docker-compose.yaml_ file, example with comments below:

```yaml
version: "3.8" # Check the latest version on docker docs
services: # Define containers within this tag
  mongodb: # indented by 2 spaces, this is the container identifier
    image: "mongo" # official image name, url, or other identifiers (we don't need to add rm or d modes because they are default)
    volumes: # List of volumes
      - data:/data/db
    environment:
      MONGO_INITDB_ROOT_USERNAME: max
      MONGO_INITDB_ROOT_PASSWORD: secret
    # In case our environment variables are in a file, we can add the following (paths are relative to the docker-compose file)
    # env_file:
    #  - ./env/the_prop_file
    # networks: # This tag is optional because with docker compose, the containers are automatically put in its own network
    #   - goals-net          
  backend: # This image was created with a docker file, so we need to instruct docker-compose how to find the Dockerfile
    build: ./backend # This will look for this path relative to the docker-compose.yaml.
    # build: # this is the longer form
    #   context: ./backend # path to the docker file
    #   dockerfile: Dockerfile # in case your file is not named 'Dockerfile', otherwise redundant
    ports:
      - "80:80"
    volumes:
      - logs:/app/logs
      - ./backend:/app # Note that we can pass relative paths
      - /app/node_modules # anonymous volume
    depends_on: # To specify dependencies between the different containers
      - mongodb
  frontend:
    build: ./frontend
    container_name: goals-react # Assigning names explicitely
    ports:
      - "3000:3000"
    volumes:
      - ./frontend/src:/app/src
    depends_on:
      - backend
    stdin_open: true
    tty: true # The combination of stdin_open and tty is the equivalent to the -it parameter on docker run
volumes: # Any named volumes declared in your services, needs to be added here (not anonymous volumes or bind mounts)
  data:
  logs:
```

Start the docker compose by running `docker-compose up (use -d for detach mode)` from the same folder your _
docker-compose.yaml_ file is, stop it using `docker-compose down (use -v to remove volumes also)`. If you
type `docker-compose up --help` you can see the options you can pass to this command, for example the `--build` parameter,
which is used to build images before starting the containers (equivalent to docker-compose build).

## Working with "Utility Containers" & Executing Commands In Containers

Utility containers are those which only have an environment in them, like a php or node environment (but no application). To
demonstrate this feature, you can execute something like `docker run -it -d node`, which would run node in interactive mode,
but also detached, so the container is still waiting for inputs. You can connect to this running container
with `docker exec <opts> <container name> <command to execute>`. With docker exec you can run commands in a running container
without interrupting the default command. You can also overwrite the default command when running a container, you do this
like `docker run <opts> <container name> <command to execute>`. To construct this utility containers, it is possible to not
add a `CMD` or `ENTRYPOINT` command to the docker file. Using this in conjunction with bind mounts, it is possible to have a
development environment without the need of installing new tools, without the inconvenience of having to install those extra
tools. What if we want to create a utility container that restricts the commands we want to allow to run? For example a node
utility container that only allows to run _npm_ commands? The answer to that question is the _ENTRYPOINT_ command. This
command is quite similar to the _CMD_ instruction, but the key difference here is that if we use _CMD_ in a docker file and
call something like `docker run <opts> <container name> <command to execute>`, the `<command to execute>` will overwrite the
CMD instruction, while with _ENTRYPOINT_ the `<command to execute>` is appended to the entry point. imagine that we have a
Dockerfile with the following content:

```Dockerfile
FROM node

WORKDIR /app

ENTRYPOINT ["npm"]
```

we will be able to execute the following to initiate the container: `docker run -it node install`, which will be the
equivalent of running `npm install` inside the container. To use this concept in a docker compose file so we can type less
stuff, we can do something like the example below:

```yaml
version: "3.8"
services:
  npm:
    build: ./
    stdin_open: true
    tty: true
    volumes:
      - ./:/app
```

And then instead of using `docker-compose up`, we can use `docker-compose exec` to run docker compose commands in running
containers, but we do have `docker-compose run <service name (npm)> <command to be appended to the entry point>`. The problem
with this approach is that as opposite of using `docker-compose down` which removes the stopped containers, this doesn't
happen when we use the `docker-compose run ...` so we need to add `docker-compose run --rm <command>`.

## Deploying Docker Containers

A container is a self-contained application with its code which can be shipped to other machines like production. Some of the
differences to consider here are:

    * In production as oppose to development, you shouldn't really use bind mounts
    * Containerized apps might need a build step
    * Multi container project might need to be split across multiple hosts
    * Trade-offs between control and responsibility might be worth it

You can run an ec2 instance on AWS and then install docker like `sudo amazon-linux-extras install docker`, then you can start
the service with `sudo servide docker start`. Once the service is started, you can start running your images there which you
can do by copying your project there and build it in ec2 or to build it locally and push the image into docker hub, and then
pull it in your ec2 instance. For that you need to execute `docker login`, create a repo in docker hub, rename your image to
match the created tag, and push it using `docker push <image>`. By default, your EC2 instance would only have the port 22
open for SSH connections, but by editing security groups, you can allow traffic to any port used by docker.In case you make
any change to your image, you need to update your image and add a new tag to your image in the docker repo, and then connect
to the ec2 instance, top the running image, and then pull the latest image and then run the image again. Obviously this "
manual" approach is very cumbersome, because you own the machine and securitization of it, you need to manually update
images, requires a lot of learning... To deal with this issues, there is another approach which is to use AWS ECS (Elastic
Container Service). This means you have to follow the approach provided by amazon to create, manage and update automatically
the containers, monitor and scale if appropriate the services which is simplified in this way.

### Multi-stage builds

There are times when you might want to have different final images depending on the environmnet you want to deploy your image
into. For example you might not want to expose a port, or to run a command like build, which might be required in lower
environments. Multi-Stage builds allow you to have one Dockerfile, that define multiple build steps or setup steps, so
called "stages" inside of that file. Stages can copy results from each other, so we can have one stage to create the
optimized files and another stage to serve them. We can either build the entire Dockerfile going through all stages, step by
step from top to bottom or we select individual stages up to which we wanna build, skipping all stages that would come after
them that are multi-stage builds and multi-stage Dockerfiles.

So imagine that you want some steps to build some optimized files, and then you want to copy those files into another base
image, but without having all the intermediate files that were required to get there (for example using node). An docker
example is shown below:

```Dockerfile
FROM node:14-alpine as build_img

WORKDIR /app

COPY package.json .

RUN npm install

COPY . .

RUN npm run build

# Now we want to use the output  of the above commands into ngnix, we can have two FROM tags and we reference to the above like this:
FROM nginx:stable-alpine

COPY --from=build_img /app/build /usr/share/ngnix/html

EXPOSE 80

CMD ["ngnix", "-g", "daemon off;"]
```

Imagine we just want to build the above image up to the point it calls the npm command, we can do that by running:
`docker build --target build_img -f <Dockerfile>`

## Getting Started with Kubernetes

[Kubernetes](https://kubernetes.io/) is a container orchestration tool independent on the cloud provider we might be using.
It tries to solve the problem of the error prone of the manual deployment, hard to maintain, handling burst loads, load
balancing and other manual deployment related issues. But mainly (because AWS already solve this type of problems), deals
with vendor lock-in issues by abstracting provisioning. The way to work with kubernetes is that we write an configuration
file, and then we can deploy our services in any machine that has the kubernetes tools and is handled by ourselves. It is
like a docker compose in multiple machines.

### Kubernetes Concepts

There are certain things that kubernetes needs in order to run a cluster. For example you have to create the kubernetes software on the nodes. Some basic concepts are:

    * Pod (Container): Smallest possible unit in kubernetes you can define and create. Host one or more application containers and their resources (Volumes, IPs)
    * Worker Node: A virtual instances with RAM and CPU. It has the following elements
        - Proxy: Element of the worker node to be configured to handle network traffic.
        - Pods: You can run one or multiple pods in a worker node.
        - Kubelet: A software service running to handle communication between the Worker Node and the Master Node.
        - kube-proxy: Service to handle incoming and outgoing traffic to ensure only allowed traffic is entering or leaving the pods.
    * Master Node: Control center/plane or machine that coordinates the worker nodes and the deployment of pods on them. Might be redundant.
        - API Server: counterpart of the kubelet, for the communication between worker and master node. 
        - Scheduler: Service responsible for watching our pods and choosing worker nodes, creating, scaling, deleting pods...
        - Kube-controller-manager: Watches and controls the worker nodes, is responsible for ensuring that we get the correct number of pods up and running.
        - Cloud-controller-manager: Cloud specific version of the kube controller manager. Translate instructions from AWS, AZURE or others.
    * Cluster: The sum of your master nodes, worker nodes and all the services in them. It translates as a set of instructions to be sent to your cloud provider

There is an extra concept to keep in mind, the Services, which are logical sets, groups of Pods with a unique Pod and Container-independent IP address. Services are important to ensure certain containers can be reached with a certain logical domain.

## Kubernetes in Action - Diving into the Core Concepts

As a developer, you need to set the infrastructure, kubernetes is not a cloud service provider or a deployment tool, it just handles the lifecycle of your application (although there are some tools like the kubermatic that can help you to set this up in some cloud providers, AWS has its own elastic kubernetes service).

To run Kubernetes we need to have a cluster, which needs at least a master node and one or more worker nodes (also requires some mandatory software). Then we have a client tool called _kubectl_, which is used to send instructions to the cluster, for example, create a cluster, add more pods and others. Refer to the [guide](https://kubernetes.io/docs/tasks/tools/) for installation steps.
For local testing, _Minikube_ can be used, which is a tool that runs a virtual machine to help you create "virtual" clusters in the air, and uses a single node cluster to handle the master and worker nodes (you still need kubectl).

As explained before, a _Pod_ is the smallest unit kubernetes can manage, it might have one or more services, and by default it has a cluster interal IP address. Pods are ephemeral, kubernetes starts, stops and replace them as needed. For pods to be managed with you, you need a controller, which is a _Deployment_ object.

### Deployment objects

The _Deployment_ objects is able to control one or more pods, the idea behind this object is that we pass a target state (desired), and kubernetes will get us there. With this object we can also pause or delete deployments and even roll them back. Deployments can also be scaled (manually or automatically).

If we use the Example7 (build the image first with `docker build -t kub-first-app .`), we can instruct kubernetes to deploy our image, first we need to check if the kubernetes minikube is running with `minikube status`, if it is not running use `minikube start --driver=virtualbox`, and you can inspect the cluster with `minikube dashboard` which will open a page in your browser with information about the cluster. To deploy the container we use _kubectl_ (check docs with `kubectl help`), and we create a deployment object `kubectl create deployment <deployment-name> --image=<image to deploy>` (You can inspect the existing deployments with `kubectl get deployments`). The previous deployment creation will fail due to image pull errors, this is because the kubernetes cluster acts like a totally different machine and won't have the image available as it needs to be on the docker registry.

To reach a pod and the container within the pod, we need a _Service_, which is responsible from exposing pods to users or external addresses, this is because Pods have internal IP addresses and they are constantly removed and redeployed. To create a _Service_ we can run `kubectl expose deployment <deployment-name> --port <port-number> --type=(ClusterIP|NodePort|LoadBalancer)`. The _LoadBalancer_ type needs to be supported by the provider (Minikube does). If we run `kubectl get services` after running the expose, we can see there is a new object of type LoadBalancer which has a EXTERNAL-IP property, which is the IP by which the service is reachable, but if we want to check a particular service, we can do `kubectl service <deployment-name>` and this will display properties (including the reachable url to the service).

To scale an application, we can submit the command `kubectl scale deployment/<deployment-name> --replicas=3` and this will set 3 replicas od the app. It is also possible to update an image with the following command `kubectl set image deployment/<deployment-name> <image-name>=<new-image-name>`, the previous command will only suceed if the new image has a different tag than the previous image, otherwise it won't. Once you issue this command, you can see the current deployment status by executing `kubectl rollout status deployment/<deployment-name>`. If for whatever reason we need to rollback a deployment, we use `kubectl rollout undo deployment/<deployment-name>`, which will undo the latest deployment, but you can also rollback to an older deployment by first, checking the history with `kubectl rollout history deployment/<deployment-name>:<optional-revision-number>`, then grabing the revision number we want to rollback to and then run `kubectl rollout undo deployment/<deployment-name> --to-revision=<revision-number>`.

### Kubernetes Declarative Approach

To run kubernetes in a declarative approach, we need to create a resource file and run the command `kubectl apply -f <path-to-yaml-file>`, in a similar way we did with the docker-compose approach. An example of the yaml file neede is shown below:

```yaml
apiVersion: apss/v1 #Check the kubernetes.io for which is the latest one 
kind: Deployment # Kind of the kubernetes object you want to deploy Deployment/Service/Job
metadata: # This is a defined structure, check the reference to see the available fields
  name: second-deployment # Up to you to set the name  
spec: # This is how the deployment is configured
  replicas: 1 # Number of replicas of this service to run
  selector: # Because the deployment is dynamic, we need to add a selector to indicate kubernetes, which of the containers defined below should it take care of
    matchLabes: # This is to match over the template/metadata/labels
      - app: second-app
  template: # This is to create a deployment
    metadata: 
      labels: 
        app: second-app # This 'app' property can be anything, it is a label to attach to this pod
    spec: # This is the specification for individual pods, as opose as the previous which was for the deployment
      containers: # Allows us to define containers which should be need as part of this pod
        - name: second-node
          image: academy/kub-first-app # Docker image
          imagePullPolicy: Always # Always pull the latest image, you can achieve the same by adding :latest on the image tag
          livenessProbe: # Check to see if your deployment is healthy or not
            httpGet: # Do a get to check the liveness of the pod, check the documentation to see the different types of livenessProbe
              path: / # Access the base path
              port: 8080 # The port can be different, in this case is the same
            periodSeconds: 3 # how often this check is done 
            initialDelaySeconds: 30 # How long kubernetes should wait to check the liveness for the first time
        # - name: other node
        #   image: other/image
```

In the case we want to declare a service yaml file, we do it similarly to the above one. An example is shown below:

```yaml
apiVersion: v1 # Service version is different to the deployment version, check the documentation for the latest one
kind: service # Type is service
metadata:
  name: backend # This is the name given to the service
spec:  
  selector: # In services, only label match is allowed, so you just need to add the labels to match to
    app: second-app # Sects all pods from the deployment with a value of app equal to second-app
  ports:
    - protocol: 'TCP' # TCP is the default, but we make it explicit
      port: 80 # Internal container port
      targetPort: 8080 # Outside port, the port that should be exposed
  type: ClusterIP # This is the default, values are the same than for the option type in the imperative command 
```

We also use this file with the command `kubectl apply -f <path-to-yaml-file-1>,<path-to-yaml-file-2>,...`. To update a deployment, simply change the value you want in the yaml file, and reapply the file with `kubectl apply -f ...`. To delete a deployment, you can use the imperative way, since kubernetes does not care how the deployment or service was created on the first place, use `kubectl delete deployment <deployment-name>` or use `kubectl delete -f=<yaml-file-1> -f=<yaml-file-2>`.

You can merge multiple files into a single one, for that, separate the declaration with three dashes, like:

```yaml
apiVersion: v1
kind: service 
# Some other definition below
--- # This is the separator
apiVersion: apss/v1 
kind: Deployment
# Some other definition below
```

In the more advanced deployment definition, aside the _matchLabel_ selector we also have the _matchExpression_ selector, a snippet is shown below:

```yaml
matchExpression: # All the expressions have to be satisfied in order to have a matching object
  - {key: app, values: [second-app, first-app], operator: in} # only a bunch of operators are allowed: in, NotIn and exists
```
You can use selectors in the delete statement like in `kubectl delete <comma-separated-type-of-resources> -l <key>=<value>`.
There is other options that can be configured within this configuration file such as environment variables, how the image should be pulled and many others. Check the [documentation](https://kubernetes.io/docs/reference/kubernetes-api/) for more details.