# FRR with OSPF

## Goal
We want to achieve connectivity between between devices on our network. It will be tested using ping. 
## Introduction
This is a guide on how to setup OSPF with FRR. I will be using docker image that I’ve built upon Ubuntus latest image. If you cannot access that image, or want to build your own, you might have to modify some steps of this guide.

The topoligy for this example is as follows.

                    192.168.1.0/24         192.168.2.0/24
         | Router 1 | --------- | Router 2 | -------- | Router 3 |
Router 1 and Router 3 will no be able to reach each other without proper routing protocol(s)

We will start by setting up the networks in docker
```sh
docker network create --subnet=192.168.1.0/24 net1
docker network create --subnet=192.168.2.0/24 net2 
```

Then create the containers and connect the networks
```
docker run --privileged -d -it --name first-router  --network net1 <img here> bash 
docker run --privileged -d -it --name second-router  --network net1 <img here> bash 
docker run --privileged -d -it --name third-router  --network net2 <img here> bash
docker network connect net2 second-router
```

The docker networks will have a gateway at the first IP address. This can be seen with 
```
docker inspect net1
docker inspect net2
```

Those IP address will not be used by our routers. The setup should now look like this.

| docker container | interface | IP address |
| ---------------- | --------- | ---------- |
| first-router     | eth0      | 192.168.1.2|
| second-router    | eth0      | 192.168.1.3|
| second-router    | eth1      | 192.168.2.3|
| third-router     | eth0      | 192.168.2.2|

Now that we have the routers up, we can test that the first and the third router can not reach each other.

```sh
docker exec first-router ping 192.168.2.2
```

If the first router can indeed ping the third router, something with the setup is wrong.

## Configure OSPF
Start by creating a shell in the first-router
```sh
docker exec -it first-router bash  
# enable the OSPF daemon; replace the ospfd ‘no’ to ‘yes’. Start FRR.
vim /etc/frr/daemons
# start the frr service 
service frr start
```

All configuration will be done in vtysh
```sh
vtysh

configure terminal
interface eth0
ip ospf area 0
exit
router ospf
exit
exit
write memory
```

Configure the second-router
```sh
docker exec -it second-router bash
# enable the OSPF daemon; replace the ospfd ‘no’ to ‘yes’. Start FRR.
vim /etc/frr/daemons
# start the frr service 
service frr start
```

All configuration will be done in vtysh
```sh
vtysh

configure terminal
interface eth0
ip ospf area 0
exit
interface eth1
ip ospf area 0
exit
router ospf
exit
exit
write memory
```

Configure the third-router
```sh
docker exec -it third-router bash
vim /etc/frr/daemons
# start the frr service 
service frr start
```
All configuration will be done in vtysh
```sh
vtysh

configure terminal
interface eth0
ip ospf area 0
exit
router ospf
exit
exit
write memory
```

## Testing
Verify that the routes are up
```sh
# in vtysh
show ip route
```
It should show routes known via OSPF, similar to this.
```sh
O>* 192.168.1.0/24 [120/2] via 192.168.1.3, eth0, weight 1, 00:03:16 
O>* 192.168.2.0/24 [120/2] via 192.168.2.3, eth1, weight 1, 00:14:26
```
Now ping will work from the first-router to the third-router
```sh
docker exec first-router ping 192.168.2.2
```
