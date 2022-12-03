# learn Linux Networking and Docker - Bridge, Virtual Ethernet and IPTables

![My Image](images/network.png)

## Network Namespaces

A namespace is a way of scoping a particular set of identifiers. Using a namespace, you can use the same identifier multiple times in different namespaces. Using network namespaces, you can create separate network interfaces and routing tables that are isolated from the rest of the system and operate independently.

To understand namespaces easily, it is worth saying Linux namespaces are the basis of container technologies like Docker or Kubernetes.

Let's create one quickly.

ip netns add ns1

Now  isolated network namespace is ns1 you created just. Now you can go ahead and run any process inside this namespace. Let's create python server inside ns1. This means that the process runs within its own network stack, separate from the host, and can communicate only through the interfaces defined in the network namespace.

 ip netns exec ns1 python3 -m http.server 8000



## Create two network namespace

    ip netns add red
    ip netns add green

## Create Veth( Virtual Ethernet) and Connect to network namespace

    ip link add rveth type veth peer name rbveth
    ip link add gveth type veth peer name gbveth

Think of VETH like a network cable.One end is attached to the host network, and the other end to the network namespace created. Let's go ahead and connect the cable, and bring these interfaces up.

## Connect cable to namespace

    ip link set rveth netns red
    ip link set gveth netns green

##  setup veth link
    ip link set rbveth up
    ip link set gbveth up

## setup loopback interface

    ip netns exec red ip link set lo up
    ip netns exec green ip link set lo up

the loopback interface directs the traffic to remain within the local system. So when you run something on localhost (127.0.0.1), you are essentially using the loopback interface to route the traffic through.

## setup  namespace interface
    ip netns exec red ip link set rveth up
    ip netns exec green ip link set gveth up


## assign ip address to namespace interfaces

    ip netns exec red ip addr add 10.10.1.10/16 dev rveth
    ip netns exec green ip addr add 10.10.1.20/16 dev gveth

## Build Bridges

A bridge is a way to connect two Ethernet segments together in a protocol independent way.

Docker has a docker0 bridge underneath to direct traffic. When Docker service starts, a Linux bridge is created on the host machine. The various interfaces on the containers talk to the bridge, and the bridge proxies to the external world. Multiple containers on the same host can talk to each other through the Linux bridge.


## create bride

    ip link add br0 type bridge

## setup bridge

    ip link set br0 up

# assign veth pairs to bridge

    ip link set rbveth master br0
    ip link set gbveth master br0

# setup bridge ip

    ip addr add 10.10.1.1/16 dev br0


Since we have the VETH pairs connected to the bridge, the bridge network address is available to these network namespaces. Let's add a default route to direct the traffic to the bridge

## add default routes for ns
    ip netns exec red ip route add default via 10.10.1.1
    ip netns exec green ip route add default via 10.10.1.1

## Done. Let's finally interact with the namespaces.

    ip netns exec red ping 10.10.1.20

## MASQUERADE

We are able to send traffic between the namespaces, but if try to send traffic outside of namespace it will fail. And for that, we'd need to use IPTables to masquerade the outgoing traffic from our namespace.

##  enable ip forwarding
    bash -c 'echo 1 > /proc/sys/net/ipv4/ip_forward'

    iptables -t nat -A POSTROUTING -s 10.10.1.10/16 ! -o br0 -j MASQUERADE

MASQUERADE modifies the source address of the packet, replacing it with the address of a specified network interface. and  it does not require the machine's IP address to be known in advance.


## How does Docker work?

Each Docker container has its own network stack, where a new network namespace is created for each container, isolated from other containers. When a Docker container launches, the Docker engine assigns it a network interface with an IP address, a default gateway, and other components, such as a routing table and DNS services


Docker offers five network types. All these network types are configured through docker0 via the --net flag

### 1. Host Networking (--net=host): The container shares the same network namespace of the default host.

# ## check the network interfaces on the host
    ip addr

###  check the network interfaces in the container

    docker run --net=host -it --rm alpine ip addr

### 2. Bridge Networking (--net=bridge/default)

#####  check the network interfaces in the container

    docker run --net=bridge -it --rm alpine ip addr

### 3. Custom bridge network (--network=xxx): 
####  create custom bridge

    docker network create foo

You can see that on the custom creation of a bridge, a bridge interface is added to the host. Now, all containers in a custom bridge can communicate with the ports of other containers on that bridge. This provides better isolation and security.

### Now let's run two containers 

    docker run -it --rm --name=container1 --network=foo alpine sh
    docker run -it --rm --name=container2 --network=foo alpine sh

#### check the network interfaces on the host

    ip addr

As expected, with bridge networking, both containers (container1, container2) have got their respective veth  cable attached.

### 4. Container-defined Networking(--net=container:$container2): 

    docker run -it --rm --name=container1  alpine sh

### 5. No networking: This option disables all networking for the container

    docker run --net=none alpine ip addr