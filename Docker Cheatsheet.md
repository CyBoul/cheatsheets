## Docker  

### Commands

```shell
docker
docker --version
docker version
docker info
docker <CMD> --help

# Listing Commands
docker image ls 
docker service ls 
docker node ls
docker stack ls
docker container ls

# Build Docker image
# Create using this directory's Dockerfile "."
docker build -t <app-name> .  													

# Execute Docker image
docker run <image-name>
docker run -p 4000:80 <appname>		# mapping port 4000 to 80			
docker run -d -p 4000:80 <appname>	# detached mode         								
# Running image
docker ps
docker ps --all
docker port <id>
docker stop <id>
docker rm <id>

# Deploy Commands
# Run the specified Compose file (.yml)
docker stack deploy -c <composefile> <appname>  								
```

---
### Images 

```shell
docker image ls 	 			# List images 
docker image ls -a   		    # List all images on this machine
docker image rm <image id>      # Remove specified image from this machine

# Remove all images from this machine
docker image rm $(docker image ls -a -q)  								 		
```

---
### Containers 

```shell
# List active containers
docker container ls 
# List container IDs only
docker container ls -q    
# List all containers, even those not running in quiet mode (only IDs)    	
docker container ls -aq		
# List all containers, even those not running									
docker container ls -a    
# Gracefully stop the specified container    								   	
docker container stop <hash>   
# Force shutdown of the specified container  								     
docker container kill <hash>  
# Remove specified container from this machine
docker container rm <hash>   
# Remove all containers
docker container rm $(docker container ls -a -q)   								
```

---
## Push to (Default: free) Repo - Pull & Run 

```shell
# Log in this CLI session using your Docker credentials
docker login        
# Tag <image> for upload to registry
docker tag <image> username/repository:tag  
# Upload tagged image to registry
docker push username/repository:tag    
# Run image from a registry
docker run username/repository:tag          								    
```

---
## Stack / Services 

```shell
# List stacks or apps
docker stack ls    
# Run the specified Compose file (.yml)                                        	
docker stack deploy -c <composefile> <appname> 
# List running services associated with an app
docker service ls     
# List tasks associated with an app           									
docker service ps <service>  
# Inspect task or container              									
docker inspect <task or container>   
# Tear down an application
docker stack rm <appname>                             							
```

---
## Swarms (Docker machines Loadbalanced Cluster)

```shell
# Initialize the swarm and put this VM as master
docker swarm init 
# View join token																
docker swarm join-token -q worker  
# Join (make others) join the swarm											
docker swarm join --token <token> <master-ip>:2377	
# View nodes in swarm (while logged on to manager)							
docker node ls   
# Take down a single node swarm from the manager            					
docker swarm leave --force      												
```

---


### Docker machines 

```shell
# Create a VM (Mac, Win7, Linux)
docker-machine create --driver virtualbox <vm-name>	
# Create a VM (Win10 Pro or Enterprise)						
docker-machine create -d hyperv --hyperv-virtual-switch "myswitch" <vm-name>
# View basic information about your node
docker-machine env <vm-name>      
# Send command through ssh to your VM      										
docker-machine ssh <vm-name> "docker node ls"       
# List VM 
docker-machine ls 			
# Get VM IP
docker-machine ip 	
# Copy file to node's home dir 
# required if you use ssh to connect to manager and deploy the app)
docker-machine scp <compose-file> <vm-name>:~ 									
```
```
docker-machine create --driver virtualbox myvm1 								# Create a VM (Mac, Win7, Linux)
docker-machine create -d hyperv --hyperv-virtual-switch "myswitch" myvm1 		# Create a VM (Win10 Pro or Enterprise)
docker-machine env myvm1                										# View basic information about your node
docker-machine ssh myvm1 "docker node ls"        								# List the nodes in your swarm
docker-machine ssh myvm1 "docker node inspect <node ID>"        				# Inspect a node
docker-machine ssh myvm1 "docker swarm join-token -q worker"   					# View join token
docker-machine ssh myvm2 "docker swarm join --token <token> <master-ip>:2377"	# Join (make others) join the swarm
docker-machine ssh myvm1   														# Open an SSH session with the VM; type "exit" to end
docker-machine ssh myvm2 "docker swarm leave"  									# Make the worker leave the swarm
docker-machine ssh myvm1 "docker swarm leave -f" 								# Make master leave, kill swarm
docker-machine env myvm1      													# show environment variables and command for myvm1
docker-machine ls 																# list VMs, asterisk shows which VM this shell is talking to
docker node ls                													# View nodes in swarm (while logged on to manager)

& "C:...\docker-machine.exe" env myvm1 | Invoke-Expression   					# Windows command to connect shell to myvm1
eval $(docker-machine env myvm1)         										# Mac command to connect shell to myvm1

docker stack deploy -c <file> <app>  											# Deploy an app; command shell must be set to talk to manager (myvm1), uses local Compose file
docker-machine scp docker-compose.yml myvm1:~ 									# Copy file to node's home dir (only required if you use ssh to connect to manager and deploy the app)
docker-machine ssh myvm1 "docker stack deploy -c <file> <app>"   				# Deploy an app using ssh (you must have first copied the Compose file to myvm1)

eval $(docker-machine env -u)     												# Disconnect shell from VMs, use native docker
docker-machine start myvm1           											# Start a VM that is currently not running
docker-machine stop $(docker-machine ls -q)               						# Stop all running VMs
docker-machine rm $(docker-machine ls -q) 										# Delete all VMs and their disk images
```

---

### Compose.yml 

```yaml
version: "3"
services:
  web:
    image: docker94628/testrepo:part2  		# username/repo:tag of a pushed build
    deploy:
      replicas: 5							# duplicate 5 times for clustering
      resources:
        limits:
          cpus: "0.1"						# limit to 10% of CPU
          memory: 50M						# limit to 50M of RAM
      restart_policy:
        condition: on-failure				# restart on failure
    ports:
      - "80:80" 	                        # outside-port:inside-port
    networks:
      - webnet
  visualizer:
    image: dockersamples/visualizer:stable  # username/repo:tag of pushed build
    ports:
      - "8080:8080" 		                # outside-port:inside-port
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
	    # constraint to be on the manager node while clustering
        constraints: [node.role == manager]	
    networks:
      - webnet
  redis:
    image: redis 							# image name of a pushed build
    ports:
      - "6379:6379" 		                # outside-port:inside-port
    volumes:
      - "/home/docker/data:/data"		
      # persit data by creating "./data" on the VM and refer it here			
    deploy:
      placement:
        # constraint to be on the manager node while clustering
        constraints: [node.role == manager]		
    command: redis-server --appendonly yes
    networks:
      - webnet
networks:
  webnet:
```

---

### Dockerfile 

```Dockerfile
# Use an official Python runtime as a parent image
FROM python:2.7-slim

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY . /app

# Install any needed packages specified in requirements.txt
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME World

# Set proxy server, replace host:port with values for your servers
# ENV http_proxy host:port
# ENV https_proxy host:port

# Run app.py when the container launches
CMD ["python", "app.py"]

```