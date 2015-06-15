# cjp-docker-compose
Docker Compose configuration with HA Proxy for CloudBees Jenkins Platform.

Uses [Docker Machine](http://docs.docker.com/machine/) and [Docker Compose](https://docs.docker.com/compose/) to build a CloudBees Jenkins Platform environment using Docker containers.

Mostly specific to Mac OS X but should work on Windows and Linux as well.

####Base includes:
- HA Proxy  - stats available at: http://{hostname}:9000
  - All CloudBees Jenkins instances are setup for HA via HA Proxy
- elastic search to use with CJOC (optional, use embedded if you want to): http://{hostname}:9200
- CJOC - availabe at http://{hostname}
  - JNLP port: `4001`
- CJE Client Master api-team: http://{hostname}/api-team/
- Client Master mobile-team: http://{hostname}/mobile-team/
- 2 linux slaves with git installed, running ssh on port 22, Jenkins Home - /home/jenkins
  - slave1
  - slave2
- 1 Docker in Docker (DIND) Jenkins slave
  - slaveDocker1
- default {hostname} is `jenkins.beedemo.local`, if you want to change this then you will have to update the `docker-compose.yml` file `ambassador` `links` mapping to match your hostname (this is required to have CJOC/CJE breadcrumbs and JNLP connectivity)

###Instructions
- install VirtualBox 4.3.26 or later
- install [Docker Machine](http://docs.docker.com/machine/#installation)
- install [Docker Compose](https://docs.docker.com/compose/install/)
- create a Docker Machine (I use beedemo-local as `{machine_name}`)
  - `docker-machine create --driver=virtualbox --virtualbox-memory=4096 {machine_name}`
  - set env for newly created machine: `eval "$(docker-machine env)"`
- replace vboxfs (VirtualBox share) /Users share with nfs share
  - Create NFS share on Mac OS X side:
    - create exports file: `sudo vi /etc/exports` with contents (IP used here is your `docker-machine ip {machine_name}`): `/Users 192.168.99.100 -alldirs -mapall={your_username}`
    - restart nfsd: `sudo nfsd restart`
  - Setup Docker Machine nfs:
    - Get IP of you VBox for the Docker host you are updating: `VBoxManage showvminfo {machine_name} --machinereadable | grep hostonlyadapter`
    - Run the following command to get the IPAddress for the VBox Network Adapter that matches the name from above: `VBoxManage list hostonlyifs`
	- ssh into machine: `docker-machine ssh {machine_name}`
    - Add a bootlocal script to your Docker Machine to start the NFS service on boot:
      - `sudo vi /var/lib/boot2docker/bootlocal.sh`
        
        ```
        #/bin/bash
        sudo umount /Users
        sudo /usr/local/etc/init.d/nfs-client start
        sudo mount -t nfs -o noacl,async 192.168.99.1:/Users /Users
        ```
    - Make the `bootlocal.sh` file executable: `sudo chmod +x /var/lib/boot2docker/bootlocal.sh`
    - exit ssh and restart Docker Machine: `docker-machine restart`
- Clone this repo anywhere under your `/Users` directory
- If you would like to store your Jenkins `HOME` directory somewhere else you need to update the `docker-compose.yml` file:
  - Update `data` under `joc1`` -> `volumes` to point to where you want your Jenkins `HOME` directory. 
  NOTE: You could have several different directories configured for different demos and just change this to point to the demo you want to run.
- VERY IMPORTANT: Update the `ambassador` `command: -name` value to use the fully qualified docker name for the proxy container - something like `{directory_container_dokcer_compose}_proxy_1`
- Add entry that maps Docker Machine IP (`docker-machine ip beedemo-local`) to the hostname you are using in /etc/hosts: `192.168.99.100  jenkins.beedemo.local`
- Start the docker containers in the Docker Compose file with: `docker-compose up -d`
- Run `docker-compose ps` to make sure all of the containers are up
- Now configure CJP:
  - connect masters
  - create slaves
  - create jobs
