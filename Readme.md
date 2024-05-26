To create a docker util image
1. Create a Dockerfile, I used only the baseimage and then set the workdir
2. Build the Dockerfile
3. Exec command `docker run -it node-util npm init` but here is a catch, using volume mount clone the execution of the command to the host machine as the idea of the docker utility is to do things without having to setup the host machine
4. Transform the command as `docker run -it -v $(pwd):/app node-util npm init`
5. There is also `ENTRYPOINT ['npm']` available for docker, which then thereby appends to the commands inserted, the above command can then be modified as `docker run -it node-util init` and would function the same as "npm init"
6. There is one downside to all of these as these commands can really long, so docker-compose comes up with a solution
7. With the docker-compose.yml file written as is, it cannot be used just like `docker-compose up` or `docker-compose up init` even though the interactive mode is enabled, hence to use the feature, we do `docker-compose run --rm <single_service_name> <commands_to_be_executed>`



TBN
I wanted to point out that on a Linux system, the Utility Container idea doesn't quite work as you describe it.  In Linux, by default Docker runs as the "Root" user, so when we do a lot of the things that you are advocating for with Utility Containers the files that get written to the Bind Mount have ownership and permissions of the Linux Root user.  (On MacOS and Windows10, since Docker is being used from within a VM, the user mappings all happen automatically due to NFS mounts.)

So, for example on Linux, if I do the following (as you described in the course):

Dockerfile -----

FROM node:14-slim
WORKDIR /app

--------

$ docker build -t node-util:perm .

$ docker run -it --rm -v $(pwd):/app node-util:perm npm init

...

$ ls -la

total 16

drwxr-xr-x  3 scott scott 4096 Oct 31 16:16 ./

drwxr-xr-x 12 scott scott 4096 Oct 31 16:14 ../

drwxr-xr-x  7 scott scott 4096 Oct 31 16:14 .git/

-rw-r--r--  1 root  root   202 Oct 31 16:16 package.json

You'll see that the ownership and permissions for the package.json file are "root".  But, regardless of the file that is being written to the Bind Mounted volume from commands emanating from within the docker container, e.g. "npm install", all come out with "Root" ownership.

-------

Solution 1:  Use  predefined "node" user (if you're lucky)

There is a lot of discussion out there in the docker community (devops) about security around running Docker as a non-privileged user (which might be a good topic for you to cover as a video lecture - or maybe you have; I haven't completed the course yet).  The Official Node.js Docker Container provides such a user that they call "node". 

https://github.com/nodejs/docker-node/blob/master/Dockerfile-slim.template

FROM debian:name-slim
RUN groupadd --gid 1000 node \
         && useradd --uid 1000 --gid node --shell /bin/bash --create-home node

Luckily enough for me on my local Linux system, my "scott" uid:gid is also 1000:1000 so, this happens to map nicely to the "node" user defined within the Official Node Docker Image.

So, in my case of using the Official Node Docker Container, all I need to do is make sure I specify that I want the container to run as a non-Root user that they make available.  To do that, I just add:

Dockerfile -----

FROM node:14-slim
USER node
WORKDIR /app

--------

If I rebuild my Utility Container in the normal way and re-run "npm init", the ownership of the package.json file is written as if "scott" wrote the file.

$ ls -la

total 12

drwxr-xr-x  2 scott scott 4096 Oct 31 16:23 ./

drwxr-xr-x 13 scott scott 4096 Oct 31 16:23 ../

-rw-r--r--  1 scott scott 204 Oct 31 16:23 package.json

------------

Solution 2:  Remove the predefined "node" user and add yourself as the user

However, if the Linux user that you are running as is not lucky to be mapped to 1000:1000, then you can modify the Utility Container Dockerfile to remove the predefined "node" user and add yourself as the user that the container will run as:

-------

FROM node:14-slim

RUN userdel -r node

ARG USER_ID

ARG GROUP_ID

RUN addgroup --gid $GROUP_ID user

RUN adduser --disabled-password --gecos '' --uid $USER_ID --gid $GROUP_ID user

USER user

WORKDIR /app

-------

And then build the Docker image using the following (which also gives you a nice use of ARG):

$ docker build -t node-util:cliuser --build-arg USER_ID=$(id -u) --build-arg GROUP_ID=$(id -g) .


And finally running it with:

$ docker run -it --rm -v $(pwd):/app node-util:cliuser npm init

$ ls -la

total 12

drwxr-xr-x  2 scott scott 4096 Oct 31 16:54 ./

drwxr-xr-x 13 scott scott 4096 Oct 31 16:23 ../

-rw-r--r--  1 scott scott  202 Oct 31 16:54 package.json

Reference to Solution 2 above: https://vsupalov.com/docker-shared-permissions/

Keep in mind that this image will not be portable, but for the purpose of the Utility Containers like this, I don't think this is an issue at all for these "Utility Containers"