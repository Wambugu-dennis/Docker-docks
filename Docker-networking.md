# Docker-networking
Docker containers basics networking explained

- # Short introduction about docker networks

In the Docker world, Network admins have a huge responsibility of understanding the network components found in virtualization platforms like Microsoft, Red Hat, etc. But, deploying a container isn’t simple; it requires strong networking skills to configure a container architecture correctly. To solve this issue, Docker Networking was introduced.

# Before understanding Docker Networking, let’s quickly understand the term ‘Docker’ first.

- [Docker](https://www.simplilearn.com/tutorials/docker-tutorial/what-is-docker) is a platform that utilizes OS-level virtual software, to help users to   develop, deploy, manage, and run applications in a Docker Container with all their library dependencies.

- Docker networking enables a user to link a Docker container to as many networks as he/she requires. Docker Networks are used to provide complete     isolation for Docker containers.

- There are 7 different types of networks we can deploy using docker containers.
- While VMs virtualizes the hardware, docker virtualizes the OS.
- ![image](https://user-images.githubusercontent.com/49392080/203761639-1ceb3808-219c-4b2c-9e74-d79c6c3635ea.png)

- so we have 

 hadware --> os(eg: ubuntu, kali) --> Docker engine --> containers(which can be runnign different OS's (ubuntu, centos, debian)


  - SETUP AND COMMANDS:
  - assuming you have the docker engine already installed
  
                	    sudo apt-get install docker.io
                	    
    - listing the current docker network 
    
              	        sudo docker network ls
              	        
    - yields: bridge, host & none as the available networks and their networks types/driver as a bridge, host, null respectively.

  # CREATING the NETWORKS
  # Network 1 -- bridge  (default)
- Pull image for the container (centos) 

                        docker pull centos    
                        
- run the container

                 sudo docker run -itd --name mycentoscon centos
                        
 - you can use the switch - -rm before -name to clean after use if it a lab
- to see if the container is running

                        docker ps (lists all running containers)
                    
- navigate to the container

                        docker exec -it mycentoscon bash

                        exit (to exit the container)

- when the container is deployed, docker automatically throws the containers into the bridge. By default, docker creates an ethernet virtual(veth) interface for each and connects it to the docker zero bridge which kind of acts as a switch. 
- Additionally, there is a virtual ethernet interface(eth0) for each container so that each container (eth0) connects to its corresponding (veth) that is automatically connected to the docker0 bridge (acting as a switch).
-  to verify that this is true, run
                        ip address show
 - the respective interfaces are now listed
 - to ascertain that they are linked to the bridge, run
 
                        bridge link
		      
  -this will list the interfaces and show that they are connected to docker zero.

   - Important to note is that the bridge also assigns IP addresses meaning it  runs DHCP
  - to verify this, run an inspection on the network using
  
                        sudo docker inspect bridge
		      
- this returns/lists the available containers and their respective ipv4 addresses & their mac Addresses in the same docker0/host network(subnet if you like).
- like every other network, it has DNS by taking the *** /etc/resolv.conf ***  file from the host (docker0 or host) and putting it on the container so essentially they are using the same DNS.
- Remember the docker0 acts as a switch, this, in turn, means that the containers can communicate with each other and the internet as well
- to verify this, jump into a container

        sudo docker exec -it mycentoscon -sh  (takes us to the centos container) 
        ping {ip address} (some other container in the same network)

     - the ping works for inter-container comms and the internet
     - why it works for the internet...
     - let's start with getting the IP route in the container by running
     
                       ip route 
                       
     - this returns the default route via the gateway of docker0
     - so how does docker0 connect the container to the internet? 
     - this is commonly known as NAT masquerade

- let's assume we have another container (webcon) running a web server eg: Nginx which by default is a website and will use port 80
- we want to reach the web server which in this case is a service being offered by the container (webcob), but thats where there's a task of manually exposing the ports so that the web server can be accessed over the internet.
    - how it's done:
    - note that this has to be done when deploying the container
    
                       sudo docker run -itd --rm -p 80:80 --name webcon nginx
                       
     - 80:80 means that exposes port 80 and matches it to the host port 80 so now it's reachable
     - running sudo docker ps shows what ports are being exposed.
     - this might seem tiresome to expose multiple ports and multiple containers but if you think about it, it adds a layer of isolation between your containers and your network (host). There are cases where you'd want the isolation to remain intact but that's a whole other topic.

    # Network 2 -- user-defined bridge
    -creating the network:
    
                      sudo docker network create netwk2
                      
    - to see if the network exists run
    
                      ip address show
                      
     - this will show the new network virtual bridge created (netwk2) with a new ip different from bridge0
     - to list the networks we run
     
		              sudo docker network ls
		              
- creating a new container inside the network, remember this time a few details have to be specified like the network name
               
			          sudo docker run -itd --rm --network netwk2 --name mycont2 ubuntu
			   
     - (you can add another container if you like)
    - checking if the virtual interfaces were created, run;
    
		              IP address show
		              
    - checking the connection of the new interfaces created and verifying they are tied to the bridge, run;
		              
		              bridge link
		              
    -inspecting the network, run;
		      
			          sudo docker inspect network2
			  
    - the created containers are displayed with their respective names, mac addresses and ipv4 addresses in the related to the network.
    
- important to note that the new network (netwk2) is completely isolated and cannot communicate with the default network we started with.
- if you try pinging a container in the neighboring network, it doesn't ping back; so network isolation.
- user-defined network provides for the cool container-to-container DNS
    eg; say for instance we have to containers in network two cont21 and cont22
     - these two can ping each other by name within that network BY NAME
     
                            sudo docker exec -it cont21
                            ping cont22
					                                                        and
					        sudo docker exec -it cont22
                            ping cont21
				
     # Network 3 - the Host
-  it's one of the default networks that were already there.
  - remember the webserver (webcon) in the first network we had, we want to redeploy that in a host network
   - first, we need to stop running that container, and run;
		      
			                sudo docker stop webcon
			  
   - steps:
    - we first define our new container and define our network 
	- we won't expose any ports for now but we will keep the same container name webcon

		            sudo docker run -it'd -rm --network host --name webcon nginx
			  
   - this now does results in a very interesting container with interesting network configuration
   - the container is now deployed and hooked to the host network directly
   - when you deploy a container to the host network, that container completely bumps off the host by sharing the same ports and ip address
   - this container runs as an application with no isolation.

   # Network 4 -- the mac vlan

- this is a very interesting network in docker.
     - remember all the networks  covered so far; what if you could do away we all that jargon of isolated networks, separation, virtual ethernets, and all, aaannnd connect the container directly to a physical network.
   - that's a mac vlan network in docker.
   - if a container is connected via a macvlan, it'd be as is the container is directly connected to a switch, have its own IP address, and even a Mac address through their own virtual ethernet interfaces.	 
       steps to creating our first mac vlan:
	   
                            sudo docker network create -d macvlan \
						    --subnet 10.2.1.0/24  \
						    --gateway 10.2.1.3 \ 
						    -o parent=denno3 \
						    (here we are trying to tie our macvlan network to the host network interface)
                             dennohmcvln

     - checking whether this network has been created, run:
		               
					        sudo docker network ls
					   
     - this shows the new macvlan just created (dennohmcvln) and the driver/network type macvlan

- remember the two containers we created in the user-defined network 2 (cont21 and cont22), we want to deploy them into the macvlan network
    steps:
    - first, we need to stop the containers
	
                            sudo docker stop cont21 cont22
					 
    - deploying then on the new macvlan network (dennohmcvln)
    (specify and assign ip addresses manually, make sure its not being used in your network and is outside your DHCP range)
	
                            sudo docker run -itd --rm --network dennohmcvln \
                            --ip 10.8.1.25 \
                            --name cont21 ubuntu
		     
 - now container cont21 is connected to the network like a regular vm downside of macvlan:
       - first, we try pinging its gateway which is its own, but nothing happens
       - this is because the network might not be able to have multiple mac addresses and the macvlan with actually shares a port with for all containers.
       - network restrictions might hinder this connection if for instance, the network restricts the port connection to a single or at most two connections per switch port.
       - the only way multiple port connection is done is by enabling promiscuous mode
       - mac address with promisc mode which you may have no control over (you always want to have control of your networks)
       - no DHCP as is usual with connecting devices directly to your home network.
       - if you don't specify IP addresses for your containers in macvlan network docker chooses one for you using its own DNCP server and assigns the way it does with bridge network.
       - the best trick here is to assign ip range the docker host should use to assign the containers
       eg: 
	   
	   	                sudo docker network create -d macvlan \
	   	                --subnet 192.168.0.0/24 \
	   	                --gateway 192.168.0.1 \
	   	                --ip-range (limit is a single ip) not very useful

      steps for enabling Promiscoius mode
      - start with the host
          
                        sudo ip link set denno3  promisc on
		     
      - next is to enable promisc mode on every container, yes very redundant
      - assuming you're on a vm (say Oracal virtual box which I used for this lab) head to settings, network promiscuous mode set to allow all
      - however, this might still not work on the first trial so reboot the host, repeat the step from ip link, and try pinging the gateway ip from a container, that should work.

    - Macvlan has two modes:
          1. Acts as a bridge network(one we just covered) except it connects to your network
          2. Has an 802.1q mode
- this mode allows connecting the containers directly to your network as well as;
    - allows specifying the sub-interface eg: eth0.30, etho.40 which enables docker to automatically create sub-interfaces and sends the individual (vlans) over a link like a truck from the container to the sub-interface created by docker.
 - creating our new macvlan with version/mode 2
 
                        sudo docker network create -d macvlan \
					    --subnet 192.168.15.0/24 \
					    --gateway 192.168.15.0  \
					    -o parent=denno3.20 dennohmcvlan2
					
 - to check if the network exists
                        ip add show
                    
    # Network 5 - IPvlan
 - this network solves the issues with macvlan that requires enabling promisc modes and all.
 - it has two modes:
       - L2 and L3
 - this network additionally allows the containers to share the MacAddress with the host but retain their IP addresses on the network; this is how the setting up of promisc mode is eliminated.
               
- L2 mode:
 - steps to setting up this network:
          
                        sudo docker network create -d ipvlan \
				        --subnet 10.10.1.0/24  \
				        --gateway 10.10.1.3  \
				        -o parent=denno3 newdennnohipvln
		     
  - adding the container to this ipvlan network
                     
		                sudo docker run -itd --rm --network newdennohipvln  \
		                -ip 10.10.1.72 \
		                --name ipcont ubuntu
        
		
   -  L3 mode:	   
  - l3 means we are done dealing with layer 2 operations that involve mac addresses and switches and now the containers only care about layer 3 operations concerned with IP addresses.
  - l3 mode containers connect to the host like a router(layer 3 connections)
  - this eliminates broadcast traffic from the network
  - notice while creating a network in this mode, you don't specify the gateway since the network connects to the host as a router
                       
		                sudo docker network create -d ipvlan \
				        --subnet 192.168.100.0/24 \
				    -o parent=denno3  \
				    -o ipvlan_mode=l3  \
				    --subnet 192.168.90.0/24 netwkipvln1
                       
- creating new containers inside the network 1 
                   
		             sudo docker run -itd --rm --network netwkipvlan1  \
				    --ip 192.168.100.10 \
				    --name ipvcont11 ubuntu
				 				 							                	and container 2
																				
                    sudo docker run -itd --rm --network netwkipvlan1  \
				    --ip 192.168.100.11 \
				    --name ipvcont11 ubuntu
                      
                       
  - creating new containers inside the network 2
                    sudo docker run -itd --rm --network netwkipvlan2  \
				    --ip 192.168.90.7  \
				    --name ipvcont21 ubuntu
				   																								and container 2
																												
                    sudo docker run -itd --rm --network netwkipvlan2 \
				    --ip 192.168.90.8  \
				    --name ipvcont12 ubuntu
				    
   - to inspect the networks of either networks created
   -  returns information on either of the networks  deployed, containers in the network, and their assigned IPs

                    sudo docker inspect {enter a network}
				   
    - Assumptions:
     - assume there are two networks(you can create a network by now) netwkipvln1(has two containers ipvcont11 and ipvcont12) anda second netwrok netwrkipvln2 (wit two containers ipvcint21 and ipvcont22)  created using above commands
      - netwkipvln1 has ip 192.168.100.0/24 and netwkipvln2 has ip 192.168.90.0/24
       - as it stands, the home network cannot reach these networks once they are deployed
        - the container connections to the internet is also isolated and they can only communicate between themselves.
         - interesting to note in this mode:
          - when separate networks share the same parent interface, they can communicate with each other
           - if you want network isolation of the two, create and connect each to a separate network interface with ipvlan l3s mode
            - to ensure that the containers have access to everything, add static routes on your router to tell your network how to reach the container networks.
       
   # Network 6 - Overlay network
   - this network is used when working with multiple hosts. the above has been demonstrated  under the assumption of one host.
   -  it is mostly used in production.
      you can go over the documentation about it[ here](https://docs.docker.com/network/)
      

   # Network 7 - None network
   - this is the most secure network in the list (haha)
   - this network exists by default and doesn't need to be created
   - the driver/network type is null
       - creating a container in the none network
                       
                       	sudo docker run -tds -rm --network non --name mynonecont ubuntu
                       
	- running the following commands to jump into the container see the configuration:
        
                       sudo docker exec -it mynoncont sh
                       
                       ip address show
                       
        - ip address show command returns the loopback address alone and nothing else
		
		
		
   *** this is the end of this lab project, please feel free to reach out with a comment, suggestion, or question. 
	Thank you ***
