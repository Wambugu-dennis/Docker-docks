# Docker-networking
Docker containers basics and the network infrastructure explained
        
- There are 7 different types of networks we can deploy using docker containers.
- While VMs virtualizes the hardware, docker virtualizes the OS.
- so we have hadware --> os(eg: ubuntu, kali) --> Docker engine --> containers(which can be runnign different OS's (ubuntu, centos, debian)


   - SETUP AND COMMANDS:
   ***assuming you have the docker engine allready installed*** 
                #sudo apt-get install docker.io
    - losting the current docker networks
                #sudo docker network ls
     yields: bridge, host & none as the available networks and their networks types/driver as bridge, host, null resepectively.
          
   # CREATING the NETWORKS
   
  # Network 1 -- bridge0  (default)
        
- Pull image for container (centos)
          #docker pull centos
    
- run the container
          #sudo docker run -itd --name mycentoscon centos
    **you can use the switch --rm before -name to clean after use if it a lab
- to see if the container is running
          #docker ps (lists all running containers)
    
- navigate to the container
          #docker exec -it mycentoscon bash  
          #exit to exit the container
    
- when the container are deployed, docker automatically throws the containers into the bridge. By default docker creates an ethernet virtual(veth) interface for each and connects it to the docker zero bridge which kind of acts as a switch. Additionally there is a virtual ethernet interface(eth0) for each container so that each cotainer (eth0) connects to its corresponding (veth) that is automatically connected to the docker0 bridge (acting as a switch).
-  to verify that this is true, run
         #ip address show
    -the respective interfaces are now listed
    -to assertain that they are linked to the bridge, run
         #bridge link
    -this will list the interfaces and show that they are connected to docker zero.
    
- Important to note is that the bridge also assigns ip addresses meaning it  runs DHCP
  - to verify this, run an inspection on the network using
         #sudo docker inspect bridge
  -this returns/lists the available containers and their respective ipv4 addresses & their macAddresses in the same docker0 network(subnet if you like).
  -like every other network, it has DNS by taking the /etc/resolv.conf file from the host (docker0) and putting it on the container so essentilaly they are using the same DNS. 
  -Remember the docker0 acts as a switch, this inturn means that the containers can communicate to each other and the internet as well
  - to verify this, jump into a container
         #sudo docker exec -it mycentoscon -sh  (takes us to the centos container)
         #ping 172.18.0.3(some other container in the same network)
        -the ping works for inter container comms and the internet
        
       -why it works for the internet...
      -lets start with getting the ip route in the container by running
         #ip route  
      -this returns the a default route via gateway of docker0
      -so how does docker0 connect the container to the internet?
      -this is commonly known as NAT masqurrade
      
      
- lets assume we have another container (webcon) running a web server eg: nginx which by default is a website and will use port 80
- we want to reach the web server which in this case is a service being offered by the container (webcob),but thats where theres a task of manually exposing the ports so that the webserver can be accessed over the internet. 
       -how its done:
       -note that this has to be done when deploying the container
         # sudo docker run -itd --rm -p 80:80 --name webcon nginx
       -80:80 means that exposes port 80 and matches it to the host port 80 so now its reachable
       -running sudo docker ps shows what ports are being exposed.
       -this might seem tiresome to expose multiple ports and multiple containers but if you think about it, it adds a layer of isolation between your containers and your network (host). There are cases where you'd want the isolation to remain intact but thats a whole other topic.
       
       
       
    # Network 2 -- user defined bridge
       
    -creating the network:
        #sudo docker network create netwk2
    -to see if the network exists run
         #ip address show
        -this will show the new network virtual bridge created (netwk2) with a new ip different from bridge0
     - to list the networks we run
         #sudo docker network ls
     
- creating a new container inside the network, remember this time a few details have to be specified like the network name
         #sudo docker run -itd --rm --network netwk2 --name mycont2 ubuntu
        ***you can add another container if you like***
    -checking if the virtual interfaces were created, run;
         #ip address show
    -checking the connection of the new intrfaces created and verify they are tied to the bridge, run;
         #bridge link
    -inspecting the network, run;
         #sudo docker inspect network2
    - the created containers are displayed with their respective names, macAddresses and ipv4 addresses in the related to the network.
    
- important to note is that the new network (netwk2) is completely isolated and cannot communicate to the default network we started with.
- if you try pinging a container in the neighbouring network, it doesn't ping back; so network isolation.
- user defined network provides for cool container to container DNS
    eg; say for instance we have to containers in network two cont21 and cont22
        these two can ping each other by name within that network BY NAME
          #sudo docker exec -it cont21
          #ping cont22
        


     # Network 3 -- the Host
     
  -  its one of the default networks that was already there.
  - remember the weserver (webcon) in the first network we had, we want to redeploy that in a host newtwork
         - first we need to stop running that container, run;
            #sudo docker stop webcon
    
      # steps:
         - we first define our new container and define our network
         - we wont expose any ports for now but we will keep the same container name webcon
             #sudo docker run -itd -rm --network host --name webcon nginx
    
      -this now does results into a very interesting container with intersting network configuration
      -the container is now deployed and hooked to the host network directly
      -when you deploy a container to the host newtork, that container completely bumps off the host by sharing the same ports and ip address
      -this container runs as an application with no isolation.
     

     # Network 4 -- the mac vlan
     
     -this is a very interesting network in docker.
     -remember all the networks we have covored so far; what if we could do away we all that jagon of isolated networks, sepaaration, virtual ethernets and all, aaannnd connect the container directly to a physical network.
     -thats a mac vlan network in docker.
     -if a container is connected via a mac vlan, itd be as is the container is directly connected to a switch, have its own ip address and even a mac address through their own virtual ethernet interfaces.
   
     #steps to createing our first mac vlan
     
             # sudo docker network create -d macvlan \
             > --subnet 10.2.1.0/24 \
             > --gateway 10.2.1.3 \
             > -o parent=denno3 \ (here we are trying to tie our macvlan network to the host network interface
             > dennohmcvln
      
        - checking whether this network has been created, run:
             #sudo docker network ls
        -this shows the new macvlan just created (dennohmcvln) and the driver/network type macvlan
       
- remember the two contanainers we created in the user defined network 2 (cont21 and cont22), we want to deploy them into the macvlan network
      # steps:
    - first we need to stop them
            #sudo docker stop cont21 cont22
    - deploying then on the new macvlan network (dennohmcvln)
    (specify and assign ip addresses manually, make sure its not being used in your network and is outside your DHCP range)
            #sudo docker run -itd --rm --network dennohmcvln \
            > --ip 10.8.1.25 \
            > --name cont21 ubuntu
         
         - now container cont21 is connected to the network like a regular vm 
         downside of macvlan:
         - first we try pinging its gateway which is its own, nothing happens
            -this is because the network might not be able to have multiple macaddresses and the acvlan swith actually shares a port with for all containers.
            - network restrictions might hinder this connection if for instance the newtork restricts port connection to single or at most two connections per switch port.
            - the only way multiple port connection is done is by enabling promiscous mode
            - mac address with promisc mode which you may have no control over (you always want to have control of your networks)
            - no DHCP as is usual with connecting devices directly to your home network.
                - if you dont specify ip addresses for your containers in macvlan network docker chooses one for you using its own DNCP server and assign the way it does with bridge network.
                
                - the best trick here is to assisgma ip range the docker host should use to assign the containers
                
                eg: sudo docker network create -d macvlan --subnet 192.168.0.0/24 --gateway 192.168.0.1 --ip-range (limit is a single ip) not very useful
                
            
                 -- steps for anabling promiscoius mode
             - start with the host
                    #sudo ip link set denno3  promisc on
             - next is to enable promisc mode on on every container, yes very redundant
             - assuming youre on a vm (say oracal virtual box which i used for this lab) head to settings, network promiscous mode set to allow all
             - however, this might still not work on the first trial so reboot the host, repeat the step form ip link and try pinging the gateway ip from a container, that should work.
             
   - Macvlan has two modes:
       1. Acts as a bridge network(one we just covered) except it connects to your network
       2. Has an 802.1q mode 
             
             
             
             
             
    
     
     
     


    
    
    
    
    
    
       
    

        
