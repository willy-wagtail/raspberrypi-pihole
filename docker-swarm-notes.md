# Docker Swarm Notes

Contents

1. [ Overview ](#1-overview)
2. [ Networking ](#2-networking)
3. [ Orchestration ](#3-orchestration)
4. [ Consistency ](#4-consistency)
5. [ Security ](#5-security)
6. [ Setting up a swarm ](#6-setting-up-a-swarm)
7. [ Managing nodes ](#7-managing-nodes)
8. [ Managing services ](#8-managing-services)
9. [ Working with stacks ](#9-working-with-stacks)

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

A web interface only available in the enterprise edition of Docker built on top of swarm mode. Provides an interface to visualise and manage clusters and manage role based access controls.

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
  - Swarm mode has an integrated service discovery system based upon DNS and is implemented in the docker engine. It is used for resolving names to IP addresses.
  - The same system is used when not running in swarm mode. Service discovery in docker is scoped to a network. The network can be an overlay (spanning multiple hosts). The same internal DNS system is used.
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

- Combines an overlay network and a service virual IP
  - manager creates an overlay network called ``ingress`` on initiation of a swarm
  - every node that joins the swarm is in the ingress network
  - when a node receives an external request, it resolves the service name to a vIP
  - the IPVS load balances the request to a service replica over the ingress network

- The nodes need to have a couple of ports open
  - port 7946 for both TCP and UDP protocols to enable container network discovery
  - port 4789 for UDP protocol to enable the container ingress network

- You can add an external load balancer on top of the load balancing provided by the routing mesh.
  - E.g. if you have nodes running on the cloud, they can be running in a private subnet on the cloud not directly accessible from the internet. You then provision a cloud load balancer to handle requests from the internet and node balance them across nodes in the swarm. The swarm nodes then load balance again across nodes running tasks for the service.

- Requires version 17.09 or greater if running Routing Mesh on Windows.

- Besides ingress network, docker creates the ``docker_gwbridge`` network automatically when you initialise a swarm or join a docker host to a swarm.
  - It's a virtual bridge that connects the overlay networks to the individual docker daemon's physical network.
  - Provides default gateway functionlity for all container attached to the network.
  - It exists in the kernel of the docker host. You can see it by listing network interfaces on your host.

## 3 Orchestration


### 3.1 Service placement

For replicated services, decisions need to be made by swarm managers for where service tasks will be scheduled or where the service is placed.

- 3 ways to influence where a service is placed: 
  - cpu/memory reservations
  - placement constraints
  - placement preferences

- global services can also be restricted to a subset of notes based on those conditions
  - a node will never have more than one task for a global service


- CPU and memory reservations can be declared
  - each service task can only be scheduled on a ndoe with enough cpu and memory
  - any tasks that remain stay in a pending state
  - global services will only run on nodes that meet a given resource reservation
  - setting memory reservations is important to avoid that container or docker daemon to get killed by the OOM killer

- Placement constrains are conditions that compare node attributes to a string value
  - built-in attributes for each node are ``node.id``, ``node.hostname``, and ``node.role``
  - engine labels are used to indicate things like OS, system architecture, avilable drivers ... e.g. ``engine.labels.operatingsystem``.
  - node labels are added for operational purposes and indicate the type of application, datacenter location, server rack, etc ... e.g. ``node.labels.datacenter``.
  - all constraints provided must be satisfied by a node in order to be scheduled a service task.

- Placement preference influence how tasks are distributed across appropriate nodes
  - currently the only distribution option is ``spread``
  - labels are used as the attribute for spreading tasks
  - multiple placement preferences can be specified and a hierarchy of preferences is created.
  - nodes that are missing a placement preference are treated as a group and receive tasks in proportion equal to all other label values.

  - placement preferences are ignored by global services

### 3.2 Update behaviour

Swarm supports rolling updates where a fixed number of replicas are updated at a time

- There are three main settings
  - update parallelism 
    - the number of tasks the scheduler updates aat a time
  - update delay 
    - the amount of time between updating sets of tasks
  - update failure action 
    - pause, continue or automatically rollback on failure

- You can set a ratio for the number of failed task updates to tolerate before failing a service and the frequency for monitoring for a failure.

### 3.3 Rolling back updates

Docker swarm mode keeps track of the previous configuration for services
- you can rollback manually at any given time, or automatically when an update fails
- the same options available for configuring update behaviour are available separately for configuring rollbacks - i.e.  rollback parallelism, rollback delay, rollback failure action.

## 4 Consistency

Swarm mode can include several manager and worker nodes that provide fault tolerance and ensure high availability.

- all managers share a consistent internal state of the entire swarm while workers do not share a view of the whole swarm
- managers maintain a consistent view of the state of the cluster by using a consensus algorithm called the Raft consensus
  - one manager is elected a leader who makes all the decisions for changing the state of the cluster towards the desired state.
  - leader accepts new service requests, service updates, and how to schedule tasks.
  - decisions are acted only when there is a ``quorum`` - when a majority of manageres agree to the proposed changes. A manager agrees by acknowledging that they have received a proposed change.
  - raft allows for ``(n-1)/2`` failures it can tolerate for the swarm to continue operating. 
    - E.g. in a swarm with 3 managers, raft allows only one manager to fail and the swarm can still operate as normal. However if two managers fail, the cluster state would freeze until a quorum of manageres are available again.
  - in the absence of a quorum, currently running services will continue to run, but no new scheduling decisions take place.
  - when a swarm is initialised, the first manager is automatically the leader. 
  - if the currently elected leader fails, or voluntarily steps down (e.g. to perform system updates), an election between the remaining manager nodes takes place. In the absence of a leader, the current state is frozen.

- manager tradeoffs: 
  - more managers, while increasing fault tolerance, will also increase the amount of managerial traffic required for maintaining a consistent view of the cluster and the time to achieve raft consensus with every state change - thus decreases performance and scalability.
  - general rules for setting the number of managers:
    - odd number of managers
    - single manager swarm is acceptable for development and test swarms
    - 3 managers can tolerate 1 manager failure
    - 5 managers can tolerate 2 manager failures
    - 7 managers can tolerate 3 manager failures
    - docker recommends a maximum of 7 managers - above 7 has too much of a performance impact.
    - distribute managers across at least 3 availability zones, in the event of a data center outage.
  
  - by default, managers perform worker responsibilities.
    - over-utilised managers can be detrimental to the performance of the swarm
    - use conservative reservations to make sure managers won't be starved for resources
    - prevent work from being scheduled on manager nodes by draining them
      - draining removes any tasks currently on a node, and prevents new tasks from being scheduled to it.

- are there worker node tradeoffs?
  - more worker nodes gives you more capacity for running services and improves service fault tolerance
  - more workers don't affect manager's raft consensus process
  - nodes participate in a weakly-consistent highly scalable gossip protocol called ``SWIM``
    - performance negligence is negligible.
  
- Raft logs are consensus state change logs recorded by the leader manager such as creating a new service. They are shared with other managers to established a quorum.
  - logs are persisted to disk in the raft subdirectory of your docker swarm data directory ``/var/lib/docker/swarm`` on linux.
  - you can backup a swarm cluster by backing up the entire swarm directory
  - you can restore a new swarm from a backup by replacing the swarm directory with the backed-up copy.

## 5 Security

cluster management

- uses pki to secure swarm communication and state
- nodes encrypts all control plane communications using mutual TLS
- manager creates several resources for security purposes
  - root certificate authority issues certificates verifying identity of certificate holders 
  - public and private key pair for secure comm between nodes in swarm
  - worker token - used by nodes to join swarm as worker. Token is a digest of the Root CA and the secret.
  - manager token - used by nodes to join swarm as managers.
- when a new node joins swarm, manager issues a new certificate identifying the node. The node uses this for communication in the swarm.
- new managers also get a copy of the root CA certificate so they can take over leadership in the event of an election
- you can use alternative CA instead of using docker to auto handle it for you
- the CA can be rotated out whenever your security policies requires it and will auto rotate the TLS certificates of all swarm nodes in the cluster

data plane

- you can enable encryption of overlay netowkrs at the time of creation
- when traffic leaves a host, an IPSec encrypted channel is used to communicate with a destination host
- the swarm leader periodically regenerates and distributes the key used for encripting IPSec data plane traffic
- overlay network encryption is not suported for windows as of docker version 17.12

raft logs / secrets
- raft logs are encrypted on the manager nodes protecting it against intruders from gaining access and protecting secrets stored in the raft logs.
- secrets allows you to securely store secreats used by services such as passwords, API keys, any other information you wouldn't want to be exposed.
- in windows, secrets are suported in verseion 17.06 and above.

locking a swarm
- by default, the keys to encrypt the logs are stored on disk along with the raft logs
- if attacker gains access to raft logs, they could gain access to the keys used to encrypt the logs
- swarm allows you to implement strategies where the key is never persisted to disk with autolock
  - swarm feature called ``autolocking``. When a swarm is autoloacked, you must provide the key when starting a docker daemon.
  - will require manual intervention when a manager is restarted.
  - can rotate the key at any point or disable autolock so managers can restart without intervention

## 6 Setting up a swarm

Options for creating a swarm

- single-node (dev and test envs only)
- multi-node clusters
  - on premises, install docker on bare metal or VMs 
    - network firewall needs to allow traffic from ports swarm mdoe requires: TCP 7946, UDP port 7946, and 4789.
  - UCP docker enterprise GUI sets all this up via the UI
  - public clouds
    - docker for azure, focker for aws, focker for ibm cloud
    - docker cloud offering

Demo notes

- single node
  - ``docker info | grep Swarm`` - "Swarm: inactive"
  - ``docker swarm --help``
  - ``docker swarm init`` - runs single node swarm
  - ``docker swarm join-token manager`` - adds managers
  - ``docker info`` - "Swarm: active". Also shows number of managerse, nodes, raft internals, etc.
  - ``docker info --format '{{json .Swarm}}'``
  - ``docker network ls`` - verify that the networks associated with swarm has been created, ``ingress`` and ``docker_gwbridge``
  - ``docker swarm leave --force`` - leave the swarm. Force is required because when the last manager leaves, all the swarm state goes with it.

- multi-node
  - ``docker-machine | more`` - docker machine comes installed in docker for mac and docker for windows, and can be used to quickly create docker-enabled VMs
  - ``docker-machine create vm1`` - creates a VM in VirtualBox by default using an image with docker already installed. VirtualBox needs to be installed as a prerequisite.
    - ``vm1`` is the name of the node. Don't name with manager or worker in name as roles within the swarm can change.
  - ``docker-machine create vm2`` - create vm2
  - ``docker-machine create vm3`` - create vm3
  - ``docker-machine ls`` to list all the VMs and their network details, and docker versions
  - ``docker-machine ssh vm1`` - to ssh into vm1
  - ``docker info`` - confirms that docker is installed and swarm is inactive
  - ``docker swarm init --help`` - list docker swarm init options, we want to use the ``advertise-addr`` option. 
    - It's the IP address other nodes wil use to join the swarm
  - ``docker swarm init --advertise-addr=192.168.99.100`` - can alternatively provide the network interface name
    - copy the prepared join ``docker swarm join`` command for joining worker nodes into the swarm. 
    - you can also retrieve this after by using the ``docker join-token swarm`` command
  - ``exit`` - leave vm1 
  - ``docker-machine ssh vm2`` - ssh into vm2 
  - ``docker swarm join --token SWMTKN-1-xxxxxxxxxxxxx`` - paste in the copied command to join the swarm as a worker
  - ``exit`` - leave vm2
  - ``docker-machine ssh vm3`` - repeat for vm3
  - ``docker swarm join --token SWMTKN-1-xxxxxxxxxxxxx``
  - ``exit``
  - ssh into manager node, vm1, and check that there are 3 nodes using ``docker info``. 

## 7 Managing nodes

- promoting worker node to manager
- demoting manager to worker
- availability - active, pause, drain
- labelling - influence where nodes are placed in the swarm

Demo notes
- ``docker-machine ssh vm1``- connect to manager node, vm1, for node management
- ``docker node --help`` - node management commands.
  - ``ls`` for listing nodes
  - ``ps`` for listing tasks
  - ``rm`` for removing nodes from swarm
  - ``inspect`` lists details about a node
- ``docker node ls`` - to get a view of the swarm
- ``docker node promote vm2`` - to promote vm2 to manager
- ``docker node ls`` - to see vm2 with a manager status of ``Reachable``. The other statuses are ``Leader`` and ``Unavailable``.
- ``docker node demote vm2`` - to demote vm2 back to worker node
- ``docker node update --help`` - to check node update options we have available to us
  - ``label-add``, ``label-rm`` to add or remove a label to the node
  - ``availability`` to set availability of the node to active, pause or drain
  - ``role`` which can update a node to worker or manager. ``promote`` and ``demote`` are a shorthand of using this option.
- ``docker node update --availability drain vm1`` - set vm1 to drain to no tasks will be assigned to it.
- ``docker node ls`` - to see vm1 with with an availability of ``Drain``.
- ``docker node update --availability active vm1`` - set vm1 to active to start getting tasks again.
- ``docker node update --label-add zone=1 vm1`` - to add a label to vm1 to say it's in zone 1
- ``docker node update --label-add zone=2 vm2`` - to add a label to vm2 to say it's in zone 2
- ``docker node update --label-add zone=3 vm3`` - to add a label to vm3 to say it's in zone 3
- ``docker node inspect vm3`` - to see node labels
  - can also filter out everything but the labels using the format option with go template: ``docker node inspect -f '{{.Spec.Labels}}' vm3``

## 8 Managing services

plan - create 2 services
- docker swarm visualizer
- nodenamer

demo notes
- ``docker service --help`` - command and options for managing services
- ``docker service create --help`` -- equivalent of ``docker run`` for swarm services
- ``docker service create --constraint=node.role==manager --mode=global --publish mode=host,target=8080 --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock --name=viz dockersamples/visualizer``
  - create swarm visualiser. Command can be shortened with stacks.
  - visualiser must run on managers, so set a constraint on ``node.role==manager``
  - for demo only, make service global so every manager runs one task for the service. it could be load balanced because the swarm state is the same regardless of which manager you use.
  - publish the port using host mode so we can compare with ingress mode
  - mount is so service container can access the manager nodes docker daemon socket, where it pulls the swarm state information from
  - name the service ``viz``
  - specify the image name 
  - after running, it'll say in the logs ``... Service converged`` to let us know the actual state has converged to the desired state in the service spec.
- ``docker service inspect``- we can see the service spec using ``inspect``
- ``docker service ps viz`` - to check that the desired state matches the current state

- on web browser, can navigate to vm1's IP address on port 8080 to view the visualiser - e.g. on 192.168.99.100:8080
  - can see the nodes, and tasks, and that there is only one viz task in the manager node, as per the constraint. If we had no rule constraints, there would be one on every node.

- ``docker node promote vm2`` - if we promote vm2 to manager, a viz service replica will be started on vm2 because the viz service is global.
- ``docker service ls`` - to see the changed replica counts
- ``docker node demote vm2`` - demote vm2 back to worker, so we should end up with only one viz replica.


- ``docker service create --constraint node.labels.zone!=1 --replicas=2 --placement-pref 'spread=node.labels.zone' -e NODE_NAME='{{.Node.Hostname}}' -p 80:80 --name nodenamer lrakai/nodenamer:1.0.0``
  - constraint so that it never runs on zone 1
  - set 2 replicas, implies the mode is replicaed (default) rather than global. 
  - managers will spread tasks across the nodes by default, so we set a preference to control how it spreads
  - set an environment variable, using go template to find the hostname of the node
  - port 80 is published on ingress mode by default
  - set name
  - specify image and verseion

- check on the swarm state in the visualiser!

- test ingress routing by sending request to port 80 on vm1, which isnt running a task for this service. 
  - You'll see that the page still loads thanks to the ingress routing mesh. 
  - Reloading a few times, you can see the nodename changing, illustrating the virtual IP load balancing of the service.
  - sending a request to vm2 gives the same behaviour

rolling update to v1.0.1
- ``docker service inspect nodenamer`` - before upgrading, inspect the service to take not of the settings for updates
  - ``Parallelism`` is 1 by default, so only one task i task is upgraded at a time
  - ``Update order: stop-first`` - only available on docker 17.05 or above. Defaults to ``stop-first`` which stops the task first before starting a new task. Other value is ``start-first`` which starts a new task first before stopping the old one.
  - ``Update delay`` which sets the delay between rolling updates - defaults to 0.

- ``docker service scale nodenamer=6`` - scales service up to 6 replicas spread across eligible nodes.
  - even if there are more than one task per node, there are no port conflicts, which would not be the casae if host mode was used for publishing the service port.

- ``docker service update --rollback-paralellism 2 nodenamer`` - sets parallel rollback to 2
- ``docker service update --image lrakai/nodenamer:1.0.0 nonamer`` - update the version to 1.0.0.
  - it'll update one task at a time
- ``docker service rolback nodenamer`` - will rollback two tasks at a time.

## 9 Working with stacks

Stacks are groups of related services that can be orchestrated and scaled together. 
- stacks are declared using a compose file of version 3 or greater and named ``docker-stack.yml`` by convention.
- there are some differences between stacks and compose
  - some options which are valid in compose are ignored in stack, most notably ``build`` and ``depends_on``. Refer to [docs](https://docs.docker.com/compose/compose-file/) to check as these change over time.
  - options available to stacks and not compose are mostly under the ``deploy:`` key.
    - ``endpoint_mode``, ``labels``, ``mode``, ``placement``, ``replicas``, ``resources``, ``restart_policy``, ``update_config``
    - currently no rollback configuration in stack file
    - placement preferences and endpoint_mode for setting virtual IP or DNS round robin service discovery are only available in version 3.3 Compose files or above
  - stack files also support ``secrets:`` at the top level

Demo notes
- ``docker service rm viz nodenamer`` - remove currently running service
- create ``docker-stack.yml`` file

  ```
  services:
    viz:
      image: dockersamples/visualizer
      deploy:
        placement:
          constraints:
            - "node.role == manager"
          mode: global
        ports:
          - target: 8080
            published: 8080
            protocol: tcp
            mode: host
        volumes: 
          - "/var/run/docker.sock:=/var/run/docker.sock"
    nodenamer:
      image: lrakai/nodenamer:1.0.1
      deploy:
        replicas: 2
        placement:
          constraint:
            - "node.labels.zone != 1"
          preferences:
            - spread: node.labels.zone
      ports:
        - "80:80"
        ...
  ```

- ``docker stack --help`` 
  - ``up``, ``down`` are aliases for ``deploy`` and ``rm``, so you can use the same commands as ``docker-compose``
- ``docker stack deploy --help`` - can prune services, control when image is resolved, and pass credentials if images in private repo
- ``docker stack deploy -c docker-stack.yml demo`` - to deploy the stack
- change the nodenamer service, adding the following invalid setting and rerun the deploy command. Each node has 1 cpu unit, so our 3 nodes only has 3 cpus, and wont have enough for all 6 replicas

  ```
  ...
  deploy:
    replicas: 6
    resources:
      cpus: '0.5'
  ...
  ```
    - swarm respects resource constraints as it deploys containers. here, it deployed 2 replicas on vm2, and 1 on vm3. Once both on vm2 are up, it stops one on vm2 so it respects the resource limitations.

- ``docker stack services demo`` - to list the services in the stack - can see that not all of the 6 replicas have been started due to resource constraints.
