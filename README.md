---
Title: Project: mwczapski/jdk8_mvn_git_docker_image

---
# Baseline Docker Image for Java Development<br>[mwczapski/jdk8_mvn_git_docker_image](https://github.com/mwczapski/jdk8_mvn_git_docker_image "Create Baseline Docker Image for Java Development")

**Description:** Create a Docker Image with OpenJDK8, Maven and Git for Java development containers
## Introduction
This document details the steps I have taken to create a Docker Image that can be used to create instances of Docker Containers ready for development of Java 8 applications or for use in CI pipeline as build workers. The image is based on the Alpine Linux base Image and includes OpenJDK8 (1.8.0_242 with musl libraries for the x64 platform) (JDK as well as JRE), Maven 3.6.3, git 2.24.1 and the OpenSSH Client.

#### References:   
[Alpine Linux baseline Docker Image](https://hub.docker.com/_/alpine "Alpine Linux baseline Docker Image")   
[Zulu OpenKDK for Alpine Linux](https://hub.docker.com/r/azul/zulu-openjdk-alpine "Zulu OpenKDK for Alpine Linux")   
[jdk8.0.242-linux_musl_x64.tar.gz Download Link](https://cdn.azul.com/zulu/bin/zulu8.44.0.11-ca-jdk8.0.242-linux_musl_x64.tar.gz "jdk8.0.242-linux_musl_x64 Download Link")  
[jdk - Zulu Community](https://www.azul.com/downloads/zulu-community/?&architecture=x86-64-bit&package=jdk "jdk - Zulu Community")  

It is assumed that the host is a Linux system used through the Windows 10 Windows Subsystem for Linux 2 (WSL2). I used this work environment with WSL2 running Debian 10. If you are working on a real Linux environment, you will need to make some adjustments to some file paths.
## Pre-requisites
### Docker
Docker for your platform is required to create Docker artefacts. It is assumed to be installed and functional.   
Please note that my Docker Desktop is installed on Windows 10 but I am using it from the WSL2 Bash shell. Some file paths used in commands are idiosyncratic. For example few commands expect semi-DOS file paths, like `d:/docker/javadev/docker`, where some others require a 'real' DOS path, like `d:\\docker\\javadev\\docker`. Keep an eye out for this is your host operating system is a real Linux/Unix.
#### References:   
[Getting Started with Docker](https://www.docker.com/get-started "Getting Started with Docker")   

## Establish the environment   
Please note that I am using the Bash shell so the command syntax and usages are from that environment.

Please note that I create and populate a bunch of environment variables so as to make it easier to make changes if needed when following the steps. I am also leaving some environment variables as comments serve as examples of what values can be provided and how they can be formatted.   

```
function toDosPath() { echo $1 | sed 's|/mnt/\(.\)|\1:|' | tr '\\' '/'; }
function toWinPath() { echo $1 | sed 's|/mnt/\(.\)|\1:|' | tr '/' '\\'; }

export JDEV_HOME=/mnt/d/docker/javadev
export JDEV_HOME_DOS=$(toDosPath ${JDEV_HOME})
export JDEV_HOME_WIN=$(toWinPath ${JDEV_HOME})
mkdir -pv ${JDEV_HOME}/docker
cd ${JDEV_HOME}/docker

export ALPMIN_USERNAME=jdev
export ALPMIN_ROOT=/${ALPMIN_USERNAME}
export ALPMIN_UID=1000
export ALPMIN_GID=1000
export ALPMIN_GNAME=${ALPMIN_USERNAME}
export ALPMIN_SHELL=/bin/ash
export ALPMIN_SHELL_PROFILE=.profile
export ALPMIN_IMAGE_VERSION=1.0.0
export ALPMIN_IMAGE_NAME=jdk8_mvn_git
export ALPMIN_CONTAINER_NAME=${ALPMIN_IMAGE_NAME}
\# export ALPMIN_ADDHOSTS=" --add-host=<hostname>:<ipaddress> "
export ALPMIN_ADDHOSTS=" "
\# export ALPMIN_SET_STATIC_IP=" --ip=<static_ip_address> "
export ALPMIN_SET_STATIC_IP=" "
export CONTAINER_VOULME_HOST="${JDEV_HOME_DOS}/${ALPMIN_USERNAME}"
export CONTAINER_VOULME_HOST_RELATIVE="../${ALPMIN_USERNAME}"
export CONTAINER_VOULME_GUEST="/home/${ALPMIN_USERNAME}"
export CONTAINER_VOULME_MAPPING=" -v ${CONTAINER_VOULME_HOST}:${CONTAINER_VOULME_GUEST} "
\# export ALPMIN_SSH_PORT_HOST=50022
\# export ALPMIN_SSH_PORT_GUEST=22
\# export CONTAINER_MAPPED_PORTS=" -p 127.0.0.1:${ALPMIN_SSH_PORT_HOST}:${ALPMIN_SSH_PORT_GUEST}/tcp "
export CONTAINER_MAPPED_PORTS=" "
export ALPMIN_NET_DC_INTERNAL=jdev_net
export ALPMIN_NET=docker_${ALPMIN_NET_DC_INTERNAL}
export ENV="/etc/profile"
export TZ_PATH=Australia/Sydney
export TZ_NAME=Australia/Sydney
export CONTAINER_ENV="/etc/profile"
export JAVA_HOME=/opt/zulu8.44.0.11-ca-jdk8.0.242-linux_musl_x64
```
Confirm the values of the environment variables.
```
clear; set | grep 'JDEV_\|ALPMIN_\|CONTAINER_\|TZ_\|JAVA_'
```
The environment variables defined above are used in the Dockerfile and in Docker commands, to ensure consistency and minimise the effort that would be required if you wanted to chage paths, object names and the like.
## Create Dockerfile
Please note that the environment variables starting with '`ENV xxxx`' in the Dockerfile will be provided values defined in the environment but will note replaced in the Dockerfile. This serves to document the values in the Dockerfile but also causes the environment variables so defined to be globally available in the docker image that will be constructed using this Dockerfile. The values of these variables will be able to be accessed inside the container if needed. 
```
cat <<-EOF > ${JDEV_HOME}/docker/Dockerfile.${ALPMIN_CONTAINER_NAME}
FROM alpine

## revise for non-alpine OS
##     shell is alpine-specific
##     .profile is ash shell-specific
##     adduser is alpine-specific
##
ENV ALPMIN_USERNAME=${ALPMIN_USERNAME}
ENV ALPMIN_UID=${ALPMIN_UID}
ENV ALPMIN_GID=${ALPMIN_GID}
ENV ALPMIN_GNAME=${ALPMIN_USERNAME}
ENV ALPMIN_SHELL=${ALPMIN_SHELL}
ENV ALPMIN_SHELL_PROFILE=${ALPMIN_SHELL_PROFILE}
ENV ALPMIN_ROOT=${ALPMIN_ROOT}
ENV JAVA_HOME=${JAVA_HOME}
ENV TZ_PATH=${TZ_PATH}
ENV TZ_NAME=${TZ_NAME}
ENV ENV=${CONTAINER_ENV}

# install packages from alpine repository
RUN apk update && \\
    apk upgrade && \\
    apk add \\
      net-tools \\
      iputils \\
      nano \\
      wget \\
      tzdata \\
      openssh-client \\
      git \\
      maven  && \\
#
# set timezone
    cp -v /usr/share/zoneinfo/\${TZ_PATH} /etc/localtime && \\
    echo "\${TZ_NAME}" >  /etc/timezone && \\
#
# add non-privileged user
    addgroup --gid \${ALPMIN_GID} \${ALPMIN_GNAME} && \\
    adduser -u \${ALPMIN_UID} -G \${ALPMIN_GNAME} -s \${ALPMIN_SHELL} -h /home/\${ALPMIN_USERNAME} -D \${ALPMIN_USERNAME} && \\
    echo \${ALPMIN_USERNAME} > pw && \\
    echo \${ALPMIN_USERNAME} >> pw && \\
    cat pw | passwd \${ALPMIN_USERNAME} && \\
    rm -v pw && \\
    mkdir -pv \${ALPMIN_ROOT} && \\
    chown -Rv \${ALPMIN_UID}:\${ALPMIN_GID} \${ALPMIN_ROOT} && \\
#
# update user profile
    echo "MAVEN_OPTS=-Xmx3g" > /home/\${ALPMIN_USERNAME}/.mavenrc && \\
    echo "export JAVA_HOME=\${JAVA_HOME}" >> /home/\${ALPMIN_USERNAME}/\${ALPMIN_SHELL_PROFILE} && \\
    echo "export MAVEN_OPTS=-Xmx3g" >> /home/\${ALPMIN_USERNAME}/\${ALPMIN_SHELL_PROFILE} && \\
    echo "export PATH=\${JAVA_HOME}/bin:\${PATH}" >> /home/\${ALPMIN_USERNAME}/\${ALPMIN_SHELL_PROFILE} && \\
    chmod u+x /home/\${ALPMIN_USERNAME}/\${ALPMIN_SHELL_PROFILE} && \\
    chown -Rv \${ALPMIN_UID}:\${ALPMIN_GID} /home/\${ALPMIN_ROOT} && \\
#
# get and install OpenJDK8
    mkdir -pv ~/Downloads && \\
    cd /root/Downloads && \\
    wget -q https://cdn.azul.com/zulu/bin/zulu8.44.0.11-ca-jdk8.0.242-linux_musl_x64.tar.gz -O zulu8.44.0.11-ca-jdk8.0.242-linux_musl_x64.tar.gz && \\
    cd /opt && \\
    tar xf /root/Downloads/zulu8.44.0.11-ca-jdk8.0.242-linux_musl_x64.tar.gz && \\
    \${JAVA_HOME}/bin/java -version && \\
    \${JAVA_HOME}/bin/javac -version && \\
    rm -v /root/Downloads/zulu8.44.0.11-ca-jdk8.0.242-linux_musl_x64.tar.gz && \\
#
# make sure that JAVA_HOME is pre-pended to the default path
    echo 'export PATH=\${JAVA_HOME}/bin:\${PATH}' >> \${ENV} && \\
#
# make sure that the container does not exit unless explicitly stopped
ENTRYPOINT ["ash","-c","while true; do sleep 10000; done"]

EOF
```
Review the Dockerfile.
```
more ./Dockerfile.${ALPMIN_CONTAINER_NAME} 	

```
## Build the docker image
```
docker build \
    --tag ${ALPMIN_CONTAINER_NAME}:${ALPMIN_IMAGE_VERSION} \
    --file ${JDEV_HOME_DOS}/docker/Dockerfile.${ALPMIN_CONTAINER_NAME} \
    --network=${ALPMIN_NET} \
    ${ALPMIN_ADDHOSTS} \
    --force-rm . \
        | tee ./${ALPMIN_CONTAINER_NAME}_${ALPMIN_IMAGE_VERSION}_image_build.log
```
## Create and Test Docker Container based on the Image
### Create network for the set of containers that will need to talk to each other
```
if [ ! $(docker network create ${ALPMIN_NET} 2>/dev/null) ]; then 
    echo "Network ${ALPMIN_NET} already exists - no need to create it"; 
fi

docker network inspect ${ALPMIN_NET}
```
### Start the image
Stop and remove old version of the container, if any.
```
docker container stop ${ALPMIN_CONTAINER_NAME} && \
docker container rm ${ALPMIN_CONTAINER_NAME} 
```

Create and start the container using the docker image created above.  
The `docker run` command below:
1. directs docker to remove the container when it is stopped
2. names the container
3. adds volume mappings (if any)
4. adds port mappings (if any)
5. publishes all ports that are defined as exposed in the Dockerfile
6. defines the hostname (which would otherwise be the container id)
7. defines the network in which the container participates as a host
8. sets a static IP address (if any)
9. detaches the container from the console when it is started
10. defines the image name and version upon which the container is to be based
```
docker run \
    --rm \
    --name ${ALPMIN_CONTAINER_NAME} \
    ${CONTAINER_VOULME_MAPPING} \
    ${CONTAINER_MAPPED_PORTS} \
    --publish-all \
    --hostname ${ALPMIN_USERNAME} \
    --network=${ALPMIN_NET} \
    ${ALPMIN_SET_STATIC_IP} \
    --detach \
        ${ALPMIN_CONTAINER_NAME}:${ALPMIN_IMAGE_VERSION}
```
Inspect the container.
```
docker container inspect ${ALPMIN_CONTAINER_NAME}
```
### Verify the running container
Verify the presence and versions of development tools, host name mapping and network interface details in the running container.
```
docker container exec \
    -itu ${ALPMIN_USERNAME} \
    ${ALPMIN_CONTAINER_NAME} \
    ${ALPMIN_SHELL} -lc '\
        echo "" && java -version && \
        echo "" && javac -version && \
        echo "" && mvn --version && \
        echo "" && git --version && \
        echo "" && cat /etc/hosts && \
        echo "" && ifconfig -a'
```
### Explore the container
As root:
```
docker container exec \
    -itu root \
    -w /root \
    ${ALPMIN_CONTAINER_NAME} \
        ${ALPMIN_SHELL} -l
```
As `${ALPMIN_USERNAME}`
```
docker container exec \
    -itu ${ALPMIN_USERNAME} \
    -w /home/${ALPMIN_USERNAME} \
    ${ALPMIN_CONTAINER_NAME} \
        ${ALPMIN_SHELL} -l
```
### Stop the container
When the container stops it will be deleted, as required by the `--rm` option on the `docker run` command used to start it. The image will remain.
```
docker container stop ${ALPMIN_CONTAINER_NAME}

docker image ls ${ALPMIN_CONTAINER_NAME}:${ALPMIN_IMAGE_VERSION}

```
## Create Docker Container using a `docker-compose` command
### Establish Environment
```
export JDEV_HOME=/mnt/d/docker/javadev
export JDEV_HOME_DOS=$(toDosPath ${JDEV_HOME})
mkdir -pv ${JDEV_HOME}/docker
cd ${JDEV_HOME}/docker

# these are source image-specific and must be left as given
export ALPMIN_USERNAME=jdev
export ALPMIN_IMAGE_VERSION=1.0.0
export ALPMIN_IMAGE_NAME=jdk8_mvn_git
export CONTAINER_VOULME_GUEST="/home/${ALPMIN_USERNAME}"

# these can be changed as required for each docker-ccompose service
export ALPMIN_CONTAINER_NAME=javadev
export ALPMIN_NET_DC_INTERNAL=javadev_net
export ALPMIN_CONTAINER_NAME=javadev
export CONTAINER_VOULME_HOST_RELATIVE="../javadev"
```
### Create a docker-compose.yml using jdk8_mvn_git:1.0.0 image
This docker-compose file defines the network that this, and related containers if any, would use to communicate, and mounts the shared host directory in the container. This shared directory can be used for bi-directional file exchange, for example to allow development on the host and compilation in the guest, where the IDE runs on the host and accesses the files in the shared directory, and compilatim and execution environment is in the container and accesses the same files form there.
```
cat <<-EOF > ${JDEV_HOME}/docker/docker-compose.yml.${ALPMIN_CONTAINER_NAME}
version: "3.1"

services:
    ${ALPMIN_CONTAINER_NAME}:
# if container name is set then docker will not be able to create multiple instances of it
# i.e. swarm or scale up
#        container_name: ${ALPMIN_CONTAINER_NAME}
        image: ${ALPMIN_IMAGE_NAME}:${ALPMIN_IMAGE_VERSION}
        # restart: always
        tty: true
        stdin_open: true
        networks:
            ${ALPMIN_NET_DC_INTERNAL}:
                aliases:
                    - ${ALPMIN_USERNAME}
        hostname: ${ALPMIN_USERNAME}
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock
# the following line can be commended out or deleted 
# if the container is not to have access to a host directory
# in which case the two environment variables used there need not be set
            - ${CONTAINER_VOULME_HOST_RELATIVE}:${CONTAINER_VOULME_GUEST}
#
networks:
    ${ALPMIN_NET_DC_INTERNAL}:
        driver: bridge
EOF
```
Inspect the generated docker-compose.yml file.
`cat ${JDEV_HOME}/docker/docker-compose.yml.${ALPMIN_CONTAINER_NAME}`
### Create and Extercise the Container
Start the container in detached mode:
```
cd ${JDEV_HOME}/docker
docker-compose -f ./docker-compose.yml.${ALPMIN_CONTAINER_NAME} up --detach ${ALPMIN_CONTAINER_NAME}
```
Determine assigned container name which, given that it is not explicitly assigned in the docker-compose.yml (commented out), will be assigned by docker. This name, or the container id, is needed to connect/attach to the container.
```
containerId=$(docker ps -aqf "name=${ALPMIN_CONTAINER_NAME}")

ASSIGNED_CONTAINER_NAME=$(docker container inspect ${containerId} | grep '"Name": "/' | sed 's|^[ ]\+"Name": "/||;s|",||')
```
Verify presence and versions of development tools, using container name or assigned container id):
```
docker container exec \
    -itu ${ALPMIN_USERNAME} \
    ${ASSIGNED_CONTAINER_NAME} \
    ${ALPMIN_SHELL} -lc '\
        echo "" && java -version && \
        echo "" && javac -version && \
        echo "" && mvn --version && \
        echo "" && git --version && \
        echo "" && cat /etc/hosts && \
        echo "" && ifconfig -a'

docker container exec \
    -itu ${ALPMIN_USERNAME} \
    ${containerId} \
    ${ALPMIN_SHELL} -lc '\
        echo "" && java -version && \
        echo "" && javac -version && \
        echo "" && mvn --version && \
        echo "" && git --version && \
        echo "" && cat /etc/hosts && \
        echo "" && ifconfig -a'
```
Connect to the container and explore, using assigned container name, container id and the output of the command that returns the id of the most recently started container.   
When you exit from the container shell the container will continue to run.
```
docker exec -itu root ${ASSIGNED_CONTAINER_NAME} ${ALPMIN_SHELL} -l

docker exec -itu root ${containerId} ${ALPMIN_SHELL} -l

docker exec -itu root $(docker ps -aqf "name=${ALPMIN_CONTAINER_NAME}") ${ALPMIN_SHELL} -l

docker exec -itu ${ALPMIN_USERNAME} ${ASSIGNED_CONTAINER_NAME} ${ALPMIN_SHELL} -l

docker exec -itu ${ALPMIN_USERNAME} ${containerId} ${ALPMIN_SHELL} -l

docker exec -itu ${ALPMIN_USERNAME} $(docker ps -aqf "name=${ALPMIN_CONTAINER_NAME}") ${ALPMIN_SHELL} -l
```
### Test bi-directional file sharing
Create a file in the user's home directory:
```
docker exec -itu ${ALPMIN_USERNAME} ${containerId} ${ALPMIN_SHELL} -lc 'ls -al ~/'

docker exec -itu ${ALPMIN_USERNAME} ${containerId} ${ALPMIN_SHELL} -lc 'touch ~/IamAFile.txt'

docker exec -itu ${ALPMIN_USERNAME} ${containerId} ${ALPMIN_SHELL} -lc 'ls -al ~/'
```
Stop the container and start it again - container does not get deleted and can be re-started, with state being preserved between "stop" and "up" commands.
```
docker-compose -f docker-compose.yml.${ALPMIN_CONTAINER_NAME} stop ${ALPMIN_CONTAINER_NAME} 

docker-compose -f docker-compose.yml.${ALPMIN_CONTAINER_NAME} up -d ${ALPMIN_CONTAINER_NAME} 
```
Verify that file has survived across re-starts.  

Please note that because the `container_name` directive in the docker-compose.yml file was commented out the container does not have a fixed name. The `docker-compose` up command will generate a container name based on the 'service' name in the docker-compose.yml file.   

To execute certain container manipulations we must find the container id and/or container name.  
While the output of the `docker-compose ps` command will show ids and names of all the running containers there are ways in which the process can be scrypted, should the need arise. The following two commands are examples of that:
```
containerId=$(docker ps -aqf "name=${ALPMIN_CONTAINER_NAME}")

ASSIGNED_CONTAINER_NAME=$(docker container inspect ${containerId} | grep '"Name": "/' | sed 's|^[ ]\+"Name": "/||;s|",||')

docker exec -itu ${ALPMIN_USERNAME} ${containerId} ${ALPMIN_SHELL} -lc 'ls -al ~/'
```
## Scale containers up and down

With the `container_name` omitted from the docker-compose.yml file it is possible to scale the number of container instances running on the host.  
Scale up to 5 container instances:
```
docker-compose -f docker-compose.yml.${ALPMIN_CONTAINER_NAME} up -d --scale ${ALPMIN_CONTAINER_NAME}=5 ${ALPMIN_CONTAINER_NAME} 

docker-compose -f docker-compose.yml.${ALPMIN_CONTAINER_NAME} ps
```
Scale back down to 1 container instance:
```
docker-compose -f docker-compose.yml.${ALPMIN_CONTAINER_NAME} up -d --scale ${ALPMIN_CONTAINER_NAME}=1 ${ALPMIN_CONTAINER_NAME} 

docker-compose -f docker-compose.yml.${ALPMIN_CONTAINER_NAME} ps
```
Stop all container instances for all services defined in the docker-compose.yml file:
```
docker-compose -f docker-compose.yml.${ALPMIN_CONTAINER_NAME} stop

docker-compose -f docker-compose.yml.${ALPMIN_CONTAINER_NAME} rm
```
## Summary
The objective of this project was to develop a Docker Image that could be used for rapidly creating Docker Container instances for Java 8 development. The image includes the OpenJDK 8, Maven and Git, which are all presumed to be needed in a Java 8 development environment, whether stand-alone or as workers in CI environments.
