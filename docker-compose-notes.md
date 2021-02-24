# Docker Compose Notes

Contents

1. [ YAML language](#1-yaml-language)
2. [ Docker compose YAML file ](#2-docker-compose-yaml-file)
3. [ Docker compose CLI ](#3-docker-compose-cli)
4. [ Building images ](#4-building-images)
5. [ Multiple environments ](#5-multiple-environments)

## 1 YAML language

- data serialization language
- .yaml or .yml file extension

### 1.1 YAML data types

#### Integers

- Examples: ``0``, ``1``, ``1000``, ``+1000``, ``-1000``

#### Strings

- quote marks are optional unless the symbol has special meaning
- use double quote marks to escape control characters

#### Null
- Examples: ``~``, ``null``

#### Booleans
- Examples: ``true``, ``false``, ``yes``, ``no``, ``on``, ``off``

- Docker Compose files limit where booleans are allowed so specifying booleans are commonly expressed as strings using quote marks.

#### Collections

- Mappings
    - ```
      key1: value1
      key2: value2
    - ```
      # Nested Mapping
      outerkey:
        innerKey: innerValue
    - ```
      # Inline Mapping
      outerKey: { innerKey: innerValue}
      ```

- Sequences
  - ```
    - value1
    - value2
  - ```
    # Nested Sequence
    -
      - innerValue
  - ```
    # Inline Sequence
    [ value1, value2 ]
    ```

- Combinations
  - Sequence in a map
    - ```
      key:
        - value1
        - value2
    - ```
      key:
      - value1
      - value2
      ```
  - Mapping in a sequence
    - ```
      -
        key1: value1
        key2: value2
    - ```
      - key: value
      ```
  
### 1.2 Comments

- Starts with ``#`` until the end of the line

## 2 Docker compose YAML file

The below are equivalent:

- ``docker run --name app-cache redis:4.0.6 -p 6379:6379 redis-server --appendonly yes``

- ```
  version: '3'
  services:
    app-cache:
      image: redis:4.0.6
      ports:
      # Use quotes as YAML parses numbers separated by colon as base-60 numbers
      - '6379:6379'
      command: 
      - "redis-server"
      - "--appendonly"
      - "yes"

      # command: ["redis-server", "--appendonly", "yes"]
  ```

Docker compose YAML files has the following root elements
  - version
  - services
  - volumes
  - networks

### 2.1 Version root element

Docker compose YAML files are versioned and the recommended version is 3 which requires Docker v1.13.0 or higher.

  - version must be a string

  - if no minor version is specified, the earliest release is used. So ``version: '3'`` implies a minor version of 0.

### 2.2 Services root element

Service root element configures services in the application consisting of nested mappings of services.

Each service can be configured with these options:

- image
- volumes
- ports
- environment
- logging
- security_opt

Some docker run commands only run in swarm mode, e.g. -m for setting memory or --cpus for setting number of cpus.

#### 2.2.1 Service dependencies

- The ``depends_on`` key

  - Specify dependencies between services using the depends_on key. 

    - Docker Compose will use this to determine the order in which to start the services.

    - Docker Compose will only begin the service processes in order, but it won't wait until the service itself has fully started up.
      - Applications should still be written such that it can tolerate connection failures.
      - If that's not possible, a script can pull the dependencies to wait until they are ready, such as ``wait-for-it.sh`` script ([see start-up order docs](https://docs.docker.com/compose/startup-order/)).

    - Example 
      ```
        depends_on:
          - redis
      ```

- The ``links`` key
  - Same as ``link`` in ``docker run``.
  - Additionally, it detemines the start-up order of services.
  - In general, networks are a better way to express communication relationships in general.

### 2.3 Volumes root element

- Volumes root element names volumes for sharing between services. Services then reference the root volume in it's service configuration.

- Volumes root element allows support for external volumes.

- Example below:
  - ``named-volume`` is a empty and is equivalent to a null datatype with value ``null`` or ``~``. Docker compose will create a volume using the default local volume driver.
  - ``external-volume`` has an element that tells Docker compose that it is an external volume.

```
version: '3' 
  services:
    app-cache:
      image: redis:4.0.6
      volumes:
      # Map host volume to container's data using the named volume
      - named-volume:/data
      # Map host directory to container's direectory using path relative to the compose YAML file
      - ./cache:/tmp/cache
      # docker will auto-create this volume inside container only
      - /tmp/stuff
  volumes:
    named-volume:
    external-volume:
      external: true
```

### 2.4 Networks root element

- Networks root element has the same function as ``docker network create`` command with a few differences.

  - By default, docker compose will create a network using the default bridge driver for an app in the docker compose file. 
  
  - The name is based on the directory of the compose file, with default appended to the end - ``<project-name>_default``.

  - All services will connect to this default network and can be reached using the service name.

  - This is different from using the ``docker run`` command with an empty network, where it'll add the container to a default network named ``bridge``.

#### 2.4.1 Example 1

- All service containers joins the default network created for the docker compose YAML file.

```
version: "3"
services:
  web:
    image: app
  cache: 
    image: redis
    ports:
      - "36379:6379"
```

- In the example above, assume that ``web`` exposes port 80.

  - ``cache`` can reach ``web`` by going to ``http://web:80``

	- ``web`` can reach ``cache`` by going to ``redis://cache:6379``

  - from the host, you can reach cache by going to ``redis://localhost:36379``

  - the host cannot reach web as no ports are published

#### 2.4.2 Example 2

- Custom networks can be declared in the docker compose root element, including external networks using ``external: true``.

- Within each service configuration, there is a ``network`` key to specify which network that service joins. You can create hostname aliases on the network as well.

```
version: '3'
services: 
  proxy: 
    image: repo/proxy
    networks:
      - frontend
  app:
    image: repo/app
    networks:
      - frontend
      - backend
  db:
    image: redis
    networks:
      backend:
        aliases:
          - database
networks:
  # uses default network config
  frontend:
  # uses external network created outside compose file
  backend:
    external: true
```

- In the example above: 
  - ``proxy`` and ``db`` are isolated from each other (It is best practice to limit communication to what is necessary).

  - ``db`` also configures the alias ``database`` for itself on the backend network so that the ``app`` service can resolve the hostname of ``db`` or ``database`` as a result. Mapping syntax is required for alias.

### 2.5 Variable substitution

- We can use shell environment variables in docker compose files by using the ``$VARIABLE`` or ``${VARIABLE}`` syntax.

- If a variable is not found, docker compose puts an empty string instead.

- One way we can set an environment variable on the host is by running ``export REDIS_TAG=4.0.5``.

```
services:
  proxy:
    image: 'redis:${REDIS_TAG}'
```

### 2.6 Extension fields

- We can reuse configuration by creating fragments. This feature only works on docker compose YAML file version 3.4 or higher.

  - We add root level keys beginning with ``x-`` and define config under it.

  - We then use yaml anchors/aliases to place the fragment whereever we need it.

    - the ``&default-logging`` anchor defines a chunk of configuration.

    - the alias ``*default-logging`` is used to refer to the chunk elsewhere

```
version: '3.4'
x-logging:
  &default-logging
  options:
    max-size: '10m'
    max-file: 7
  driver: json-file
services:
  web:
    image: repo/app
    logging: *default-logging
  cache:
    image: redis
    logging: *default-loggin
```

## 3 Docker compose CLI

### 3.1 Features of the CLI

- Can run multiple isolated environments on a single host

- Uses parallel execution model where it can for creation and execution tasks.

  - Not everything is run in parallel, such as when service dependencies are specified.

- Compose YAML file change detection caches containers when they are started so that when you ``restart`` a compose application, docker compose CLI will reuse any containers in the configuration that hasn't changed

  - You can force it to rebuild rather than use the cached container

- Open source software

### 3.2 Installing docker compose
	
- For Mac 
  - included in docker for mac and docker toolbox

- for Windows
  - docker for windows and docker toolbox includes compose
  - native windows docker daemon requires a docker-compose install separately from github

- For linux	
  - docker compose is included in many distros package manager
    sudo (yum|dnf|apt) install docker-compose
  - if not get from github

### 3.3 Docker compose CLI usage

``docker-compose [options] [command] [args]``

- When running docker compose CLI commands, it will connect it to the docker daemon on the host by default.

  - you can connect to remote hosts docker daemon instead with the ``-H`` option

  - you can also use secure remote connections with ``--tls`` options. Though thsi requires that the remote host has it's docker daemon configured to use TLS.

- Docker compose CLI looks for ``docker-compose.(yml|yaml)`` in the same directory you run the command.

  - use ``-f`` option to specify alternative location

- Uses docker compose projects to represent isolated apps

  - Each project is given a name, which is by default the directory name. This name appears in the resoures created such as networks and containers.

    - you can use ``-p`` option to specify a custom project name.

- Many commands are familiar docker commands such as:

  - ``build``, ``config``, ``create``, ``events``, ``exec``, ``images``, ``kill``, ``logs``, ``pause``, ``port``, ``ps``, ``pull``, ``push``, ``restart``, ``rm``, ``run``, ``start``, ``stop``, ``top``, ``unpause``, ``version``

- Two new commands are introduced by docker compose:

  - ``up``
    - creates networks and named volumes
    - builds, creates, starts, and attaches the service containers
      - use the option ``-d`` to let the containers run in detached mode
    - performs change detection

  - ``down``
    - a partial reversal of ``up``
    - it removes service containers, named and default networks
    - but it does not delete the created volumes and images
  
- To get information about docker compose and it's commands, use the ``--help`` command like so:

  - docker-compose --help | more
  - docker-compose up --help | more

- You can check whether a docker compose YAML file is valid by running ``docker-compose -f <compose file path> config``

## 4 Building images

- Building images with docker compose uses Dockerfiles for build instructions.

- In the docker compose YAML file, use the ``build`` key under in each service's configuration.

- Using ``docker-compose up`` command or ``docker-compose build`` will run the build for each service where there is a ``build`` key in the docker compose YAML.

  - ``docker-compose up`` will build any services where an image doesn't exist. Subsequent runs will not build a new image unless ``--build`` option is specified.

  - ``docker-compose build`` will build or rebuild images without starting a container for it

    - use option ``--no-cache`` to prevent using any image layer caching so all layers will be rebuilt.

    - use option ``--pull`` to always pull the base image described in the Dockerfile.

- The ``build`` key is used in two ways:

  - ```
      # path to the docker file
	  build: ./dir
  - ```
    # image: <custom-name:tag>
  	build:
      # path to dockerfile
      context: ./dir
      # optional - the name of dockerfile to use, default Dockerfile
      dockerfile: the.dockerfile
      # optional args - dockerfile needs to have corresponding arg instructions
      args: 
        buildno: 1
    ```

- The built image name by default is the name of the docker compose project followed by the service name - e.g. ``<project-name>_<service-name>``. Unless stated otherwise, the default tag is ``latest``.

  - to set a custom name and/or tag, specify ``image: <name:tag>`` of the service.

## 5 Multiple environments

### 5.1 Multiple compose files

  - You can keep separate compose files

  - But docker compose allows you to combine multiple compose files using a base file, and overrides file.

    - default files are ``docker-compose.yml`` and ``docker-compose.override.yml``
    - use ``-f`` option to specify non-default overrides

  - use ``config`` to view the resulting configuration. It is useful for writing and debugging multiple compose files.

  - ``-H`` option to connect to remote hosts
  
- e.g. ``docker-compose -f docker-compose.yml -f dev.docker-compose.yml config``

#### 5.1.1 Example

``Dockerfile``
  ```
  FROM python
  ADD . /src
  WORKDIR /src
  RUN pip install -r requirements.txt
  CMD ["python", "app.py"]
  ```

``docker-compose.yml`` for production
  ```
  version: '3'
  services:
    web:
      build: .
      ports:
        - '5000:5000'
    redis:
      image: redis
  ```

``dev.docker-compose.yml`` for development
  ```
  version: '3'
  services:
    web:
      volumes:
        - .:/src
  ```

  - Development overrides can include using a volume to mount the source code in your local machine to the container's ``/src`` directory. With this override, the source code can be modified, and the changes will be reflected in the running container.

  - It isn't always possible to use a common Dockerfile for each environment you wish to use the container. In that case we can override the Dockerfile for each environment (e.g.  ``(prod | dev).dockerfile`` files), then using the  ``builds`` key to point to the respective dockerfile in the respective ``(prod | dev).docker-compose.yml`` files.

    - e.g. 
      ```
        build: 
          context: .
          dockerfile: dev.dockerfile
      ```

### 5.2 Production considerations

- Remove any volumes for source code. The code should be frozen inside a production image.

- Use ``restart: always`` in your docker compose YAML file, so services will automatically restart if the container exits.

- Don't specify specific host ports. This avoids conflicts which prevents docker from bringing up a container by letting docker choose which host ports to use. You can do this by only specifying the container port in the ``ports`` sequence.

- Use production mode ``environment`` variables to configure your application.

- Consider additional services that might be useful in production such as log aggregation and monitoring services.
