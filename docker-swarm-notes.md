# Docker Compose Notes

Contents

1. [ Overview ](#1-overview)
2. [ Networking ](#2-networking)

## 1 Overview

### 1.1 Why?

- Once apps reach a certain level of complexity or scale, you use several machines running their own Docker Engine with their own containers.

- Container orchestration manages multiple container hosts in concert. Docker swarm is one such tool.

- Swarm mode is built into the docker engine and, when enabled, provides native container orchestration.

- Features include:
  - Integrated cluster management within docker engine

  - Declarative service model

  - Desired state reconcilation

  - Certificates and cryptographic tokens to secure the cluster

  - Container orchestration features
    - Service scaling

    - Multiple host networking

    - Resource-aware scheduling

    - Load balacing

    - Rolling updates

    - Restart policies

### 1.2 Two different solutions

Docker has two cluster management solutions, and terminology may be confusing:

  - Docker swarm standalone
    - The first container orchestration project by Docker

    - Uses Docker API to turn a pool of Docker hosts into a single virtual Docker host using a proxy system

    - Now referred as Docker Swarm standalone in documentation
  
  - Docker swarm mode (swarmkit)
    - Built into the Docker engine since version 1.12

    - Docker generally recommends swarm mode

These notes deals exclusively with docker swarm mode.

### 1.3 Concepts and terminology

- Swarm 
  - One or more Docker Engines running in swarm mode

- Node
  - Each instance of the Docker Enginer in the swarm

  - Possible to run multiple nodes on a single machine, e.g. using VMs

  - A node is either a manager or a worker

- Managers
  - Responsible for accepting specifications from users and drive the swarm to the specified desired state. They do so by delegating work to workers

- Workers
  - Responsible for running the delegated work

  - Workers also run an agent that reports back to the managers on the status of their work

- Service
  - Specifications that users submit to managers

  - Declares its desired state including networks, volumes, number of replicas, resource constraints, and other details.

  - Managers makes sure the actual state of the swarm matches the desired state if it is possible to realised.

  - There are two types of services

    - Replicated service
      - The number of replicas for a replicated service is based on the desired scale.

    - Global service
      - Allocates one unit of work for each node in the swarm
      - Can be useful for monitoring services

- Tasks
  - Units of work delegated by managers to realise a service configuration

  - Tasks correspond to running containers that are replicas of the service

  - Managers schedule tasks across nodes in the swarm. If a node leaves the swarm, the task that the node was running will be rescheduled between the remaining nodes in the swarm.

  - By default, managers can also run tasks like workers allowing you to have a single node swarm.
    - However, managers can be set to only schedule tasks, which may be desirable in production.

### 1.4 Universal control plane (UCP)

A web interface only available in the enterprise edition of Docker built on top of swarm mode. Provides an interface to visualise andmanage clusters and manage role based access controls.

## 2 Networking

- In swarm mode, services need to communicate with one another and the replicas of a service can be spread across multiple nodes.

### 2.1 Overlay networks

- A multi-host network in swarm is natively supported with the overlay network driver with no need for any external configuration.

-  Overlay networks only apply to swarm services and you can attach a service running in the swarm to one or more overlay networks. Containers not part of a swarm can not access overlay networks.

- Managers will automatically extend overlay networks to nodes in the swarm that run tasks requiring access.

- Network isolation and firewall rules apply to overlay networks just as they do for bridge networks.

  - Containers within a Docker network have access on all ports in the same network.

  - Access is denied between containers that don't share a common network.

  - Traffic originating inside of a Docker network and not destined for a Docker host is permitted, e.g. access to the internet.

  - Traffic coming into a docker network is denied by default. Ports must be published in order to grant access to traffic from outside docker.

### 2.2 Service discovery and load balancing

- A service discovery mechanism is required in order to connect to the nodes running tasks for a service.

  - Swarm mode has an integrated service discovery system based upon DNS and is implemented in the docker engine. It us used for resolving names to IP addresses.

  - The same system is used when not running in swarm mode. Service discovery in docker is scoped to a network. The network can be an overlay (spanning multiple hosts), but the same internal DNS system is used.

  - All nodes in a network store corresponding DNS records for the network. Only service replicas in the network can resolve other services and replicas in the same network by name.

  - Each task (or container) of a given service is discoverable with a name to IP mapping in the internal DNS. 

    - Because services can be replicated across multiple nodes, docker assigns a service a single virtual IP (VIP) and requests for the VIP address are automatically load balanced across all the healthy tasks spread across overlay network.

    - By using a VIP, clients are sheltered from the internal load balancing as they just to interact with a single IP address. Docker manages the load balancing for you since services can scale and tasks can change the node that they are scheduled on.

    - Request -> Service Discovery -> Service VIP -> IPVS Load Balancing -> Individual Task (Container)

- Example

  - Two services deployed in a swarm: service A with one replica, and service B with two replicas.

  - When service A makes a request for service B, the VIP of service B is resolved by the DNS server, and service A uses that VIP to make a request to service B.

  - Using support for IP virtual servers (IPVS), the request for the VIP address is routed to one of the two nodes running service B tasks.


- Besides the default vIP, load balancing can be configured using DNS round robin (RR) on a per service basis. 

  - When DNS round robin is used, the docker engine's DNS server resolves the server name to an individual task IP address by cycling through the list of IP addresses of nodes. 
  
  - If you need more control, DNS RR can be used for integrating you own external load balancer.

### 2.3 External access

- In swarm mode, there are two modes for publishing ports

  - Host mode

    - The same as when publishing a port when not in swarm mode where the container's port is published on the host that is running the task for a service.

    - If you have more tasks than available hosts, tasks will fail to run because the host port can only be bound to one task.

    - You can omit a host port to allow Docker to automatically assign an available port number in the default port range of ``30000`` to ``32767``. 

      - More difficult to work with and no load balancing unless you configure it externally.

    - Useful if you have an external service discovery service, and potentially for global services, where one task for a service runs on each node. 
    
      - E.g. the global service that monitors each node's health should not be load balanced since you want to get the status of the specific node.

  - Ingress mode - default service port publishing mode.

    - This mode load balances a published port across all tasks of a service. All nodes in the swarm publish the port. 
    
      - This is different from host mode where the node only publishes the port if it is running a task for the service.

    - Requests are round robin load balanced across the healthy instances of the service's tasks regardless of the node that receives the request.

    - This mode is ideal when you have multiple replicas of a service and need to load balance between them.

  
### 2.4 Routing Mesh


