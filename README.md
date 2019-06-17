## Exercises
1. Fix DTR pushes
	- Problem: Images cannot be pushed to DTR.

2. Fix an unhealthy Worker node
	- Problem: Worker node is joined, but has become unhealthy now.

3. Fix an application that cannot run
	- Problem: Application cannot find a suitable node to deploy on

4. Use best practices to fix / update Jenkins CICD system
	- Problem: Jenkins is setup, but is not able to deploy any application to the cluster


## Setup (to Break)

1. Fix DTR pushes

Install HA DTR, with a local volume mount to store images. This can be accomplished by installing in default mode & then adding replicas.

2. Fix an unhealthy Worker node

On a joined worker node, delete the docker socket & create a directory there instead and stop the docker daemon

```
sudo rm /var/run/docker.sock
sudo mkdir /var/run/docker.sock
sudo systemctl stop docker
```

3. Fix an application that cannot run on the cluster

Set the orchestrator type of all worker nodes to `kubernetes`. Running a swarm service will fail. Ensure the checkboxes in Admin Settings -> Scheduler are off.

Running any service like `docker service create nginx` will fail.


4. Use best practices to fix / update Jenkins CICD system
	Setup jenkins using the following compose:
	
```
version: "3.3"

services:
  jenkins:
    image: jenkins:latest
    user: root
    working_dir: /var/jenkins_home
    deploy:
      placement:
        constraints:
         - node.labels.role == jenkins
    ports:
     - target: 8080
```	

## Solutions

1. Fix DTR pushes

Run a `reconfigure` on DTR to configure distributed storage. Needs NFS setup.
Have a standard NFS server setup, but do not mention it to the participants

```
docker run -it --rm docker/dtr reconfigure \
--nfs-storage-url <...> \
--ucp-insecure-tls 
```

2. Fix an unhealthy Worker node

Startup docker daemon & address any issues with docker starting. You can see errors in starting docker using `journalctl -fu docker`. The socket will be corrupt or missing. 
Delete the file `/var/run/docker.sock` 

```
sudo rm /var/run/docker.sock
sudo systemctl start docker
```

3. Application needs a k8s enabled node. Change a swarm node to be kubernetes or mixed.

```
docker node update <node_id> --label-add com.docker.ucp.orchestrator.swarm=true
```

4. Many solutions to this. 
   - Configure Jenkins to use the UCP client bundle & use that to deploy applications / stacks
       - Create a user `jenkins` and assign appropriate permissions
   - Assign label `jenkins` to one of the worker nodes
   - Bind mount socket, docker binary & other tmp locations
   - Avoid `:latest` tag
   - Use resource limits & reservations
   - Use host published port
   - Set `JAVA_OPTS` as an environment variable
 

An example solution that works but may not be complete.

```
version: "3.7"

services:
  jenkins:
    image: jenkins/jenkins:slim
    user: root
    working_dir: /var/jenkins_home
    deploy:
      placement:
        constraints:
         - node.labels.role == jenkins
    environment:
      JAVA_OPTS: "-Xms2048m -Xmx2048m -XX:MaxPermSize=512m"
    ports:
     - target: 8080
       published: 39999
       mode: host
    volumes:
     - type: bind
       source: /var/run/docker.sock
       target: /var/run/docker.sock
     - type: bind
       source: /opt/docker
       target: /usr/bin/docker
     - type: bind
       source: /var/tmp/jenkins
       target: /var/jenkins_home
     - type: tmpfs
       target: /tmp
     - type: tmpfs
       target: /root
```