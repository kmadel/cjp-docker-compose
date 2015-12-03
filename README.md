# cjp-docker-compose
Docker Compose configuration with HA Proxy for CloudBees Jenkins Platform.

Uses [Docker Machine](http://docs.docker.com/machine/) and [Docker Compose](https://docs.docker.com/compose/) to build a CloudBees Jenkins Platform environment using Docker containers.

Mostly specific to Mac OS X but should work on Windows and Linux as well.

####Base includes:
- HA Proxy  - stats available at: http://{hostname}:9000
  - Only CloudBees Jenkins Operations Center instances (joc1 and joc2) are setup for high availability via HA Proxy
  - This is to demonstrate HA Proxy configuration for CloudBees HA - see entries for HTTP, JNLP and SSH
  - NOTE: Considering that both CJOC containers are deployed to the same Docker Host, HA doesn't make a lot of sense other than as an example
- CJOC - availabe at http://{hostname}
  - Preconfigured JNLP port: `4001`
  - You may override JNLP port with the `JENKINS_JNLP_PORT` environment variable, but you would need to modify the HA Proxy configuration as well
  - Preconfigured SSH port: `2021` - connect via http://{hostname}:2021 - HA Proxy is configured to forward to correct container
- Client Masters (need to be manually attached to CJOC - see [CJOC Client Masters Documentation] (http://documentation.cloudbees.com/docs/cjoc-user-guide/_managing_cloudbees_jenkins_operations_center_tasks.html))
  - CJE Client Master URL for api-team: http://{hostname}/api-team/
    - Preconfigured SSH port: `2022` - connect via http://{hostname}:2022 - HA Proxy is configured to forward to correct container
  - CJE Client Master URL for mobile-team: http://{hostname}/mobile-team/
    - Preconfigured SSH port: `2023` - connect via http://{hostname}:2023 - HA Proxy is configured to forward to correct container
- 1 linux slave with git installed, running ssh on port 22, Jenkins Home - /home/jenkins
  - URL: slave1
  - /home/jenkins mapped to host volume ./data/slave1
- 1 Docker in Docker (DIND) Jenkins slave
  - URL: slaveDocker1
  - /home/jenkins mapped to host volume ./data/slave1
- default {hostname} is `jenkins.beedemo.local`, if you want to change this then you will have to update the `docker-compose.yml` file so the `proxy` `container_name` matches your hostname (this is required to connect CJE client masters to CJOC), you also need to change the `JENKINS_URL` environmental variable for joc1 and joc2 so it matches your {hostname}

###Instructions
- install [Docker Toolbox](https://www.docker.com/docker-toolbox)
   Note: Docker Toolbox by default will create a docker-machine. Recommend that you stop the default and create a new machine with virtual memory allocated to it. 
  - current configuration requires Docker Compose version 1.5+
  - Requires Docker 1.9+ - using networking, built in bridge driver
- create a Docker Machine (I use beedemo-local as `{machine_name}`)
  - `docker-machine create --driver=virtualbox --virtualbox-memory=4096 {machine_name}`
  - set env for newly created machine: `eval "$(docker-machine env)"`
- Mac OS X ONLY: replace vboxfs (VirtualBox share) /Users share with nfs share [optional - but increases performance on Mac OS X]
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
- Add entry that maps Docker Machine IP (`docker-machine ip beedemo-local`) to the hostname you are using in /etc/hosts: `192.168.99.100  jenkins.beedemo.local`
- Start the docker containers in the Docker Compose file with: `docker-compose --x-networking up -d`
- Run `docker-compose ps` to make sure all of the containers are up 
- Go to http://jenkins.beedemo.local in your browser and enter license information or evaluation
- Now configure CJP:
  - connect masters, nothing to configure for CJOC, just create a new Client Master job
  - create slaves
  - create jobs

###Use of Docker Networking
A combination of the Docker Compose `container_name` option and the built in networking support introduced with Docker 1.9, allows for a completely generic HA Proxy configuration.  Previously the [ambassador pattern](https://docs.docker.com/engine/articles/ambassador_pattern_linking/) had been used with the Docker/Compose `links` feature to link the CJE client master containers to the CJOC HA cluster via the HA Proxy.  This was necessary because of limitations with the `links` feature - links are one-way, you can't link to a container that is already linked to another continaer - that is if `proxy` is linked to `apiTeam` container, then `apiTeam` can't link to `proxy`, thus the `ambassador` container.  Docker networking allows bi-directional linking, independed of the start-up order.  The trick in this configuration is to override the container name of the `proxy` container, allowing the necessary back-channel communication between CJOC and client masters to use the same URL (jenkins.beedemo.local) as specified for external access to CJOC.
