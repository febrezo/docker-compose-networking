Docker Networking
=================


Sample Services
---------------

This demo shows how to connect services run from different docker-compose files.
The structure of each service includes three different folders representing three different docker-compose files under three different folders.

The important issue here is the fact that Docker networks created by docker-compose are created using the name of the folder in which the `docker-compose.yml` file lives. 
Default networks are created for the different `docker-compose up` commands so a shared network has been created and shared between the different servers.

Each server contains the following files:

- `docker-compose.yml`. The `docker-compose` file that exposes the services. Each service is exposed outside its corresponding Docker image at port `1500x` where `x` is the number of the server. 
- `Dockerfile`. A file that simply prepares the Docker image.
- `server.py`. A simple Flask web server that shows the number of the server. It runs it locally in port `500x` where `x` is the number of the server.

Requirements
------------

To deploy this PoC, check that you have docker and docker-compose installed in your computer.

```
$ docker --version
Docker version 18.09.0, build 4d60db4
$ docker-compose --version
docker-compose version 1.23.1, build b02f1306
```

Once done, clone the repository using git clone and cd into it.

```
$ git clone https://github.com/febrezo/docker-compose-networking
$ cd docker-compose-networking
$ tree
.
├── AUTHORS.md
├── COPYING
├── dir1
│   ├── docker-compose.yml
│   ├── Dockerfile
│   └── server.py
├── dir2
│   ├── docker-compose.yml
│   ├── Dockerfile
│   └── server.py
├── dir3
│   ├── docker-compose.yml
│   ├── Dockerfile
│   └── server.py
└── README.md

3 directories, 12 files
```

Deploying the Environment
-------------------------

After doing `cd` into the folder, build and start the first server.

```
$ cd dir1
$ docker-compose build
...
$ docker-compose up
Creating network "dir1_some-net" with driver "bridge"
Creating container1 ... done
Attaching to container1
container1 |  * Serving Flask app "server" (lazy loading)
container1 |  * Environment: production
container1 |    WARNING: Do not use the development server in a production environment.
container1 |    Use a production WSGI server instead.
container1 |  * Debug mode: off
container1 |  * Running on http://0.0.0.0:5001/ (Press CTRL+C to quit)
```

Afterwards, we are starting a second node.

```
$ cd ../dir2
$ docker-compose build
...
$ docker-compose up
Creating container2 ... done
Attaching to container2
container2 |  * Serving Flask app "server" (lazy loading)
container2 |  * Environment: production
container2 |    WARNING: Do not use the development server in a production environment.
container2 |    Use a production WSGI server instead.
container2 |  * Debug mode: off
container2 |  * Running on http://0.0.0.0:5002/ (Press CTRL+C to quit)
```

Finally, we are starting a third node. Note that the service in the `dir3/docker-compose.yml` is named `server1` to *force* a collusion.

```
$ cd ../dir3
$ docker-compose build
...
$ docker-compose up
Creating container3 ... done
Attaching to container3
container3 |  * Serving Flask app "server" (lazy loading)
container3 |  * Environment: production
container3 |    WARNING: Do not use the development server in a production environment.
container3 |    Use a production WSGI server instead.
container3 |  * Debug mode: off
container3 |  * Running on http://0.0.0.0:5003/ (Press CTRL+C to quit)
```

From another terminal, we can check whether the service is reachable OUTSIDE the Docker container.

```
$ curl localhost:15001
Hello World from node 1!
$ curl localhost:15002
Hello World from node 2!
$ curl localhost:15003
Hello World from node 3!
```

Docker Networking
-----------------

After creating the environment, we can check the different networks created using `docker network ls`. 
Note that the name of the network created in `dir1/docker-compose.yml` (in our case, `some-net`), is preceded by `dir1_` which is the name of the folder of the docker-compose file that defined the network.

```
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
3abf72c9cd6d        bridge              bridge              local
0f1aee6f822a        dir1_some-net       bridge              local
1410fa738d4b        host                host                local
a642878f65d3        none                null                local
```

The aforementioned `dir1_some-net` network should be declared on each of the other `docker-compose.yml` files.

```
version: '3.5'

services:
  server2:
    ...
    networks:
      - dir1_some-net
networks:
  dir1_some-net:
    external: true
```

We can now login into either one of the previous machines. 
Note the names of the machines used (`server1`, `server2` AND `server1` again).

```
$ docker exec -it container1 bash
root@9453a4be6d60:/opt/app# curl server1:5001
Hello World from node 1!
root@9453a4be6d60:/opt/app# curl server2:5002
Hello World from node 2!
root@f2bb4d587a8b:/opt/app# curl server1:5003
Hello World from node 3!
```

We can get the same behaviour from other nodes:

```
$ docker exec -it container3 bash
root@428fba281b01:/opt/app# curl server1:5001
Hello World from node 1!
root@428fba281b01:/opt/app# curl server1:5002
Hello World from node 2!
root@428fba281b01:/opt/app# curl server1:5003
Hello World from node 3!
```

Lessons Learned
---------------

Some funny have been detected and are addressed below:

- As said before, the networks created use the name of the folder in which the `docker-compose.yml` file live. This can lead to some misunderstandings when deploying that may take too much time notice. Be advised.
- If the name of the different services included in different `docker-compose.yml` files are the same, Docker will transparently try to find alive services on different machines. For instance, if two servers are named myservice but one exposes a service in port 5000 and another one in port 6000, both services will be reachable INSIDE the Docker images using `curl myservice:5000` and `curl myservice:6000` in both machines. If the port is also de same, each machine will only be able to connect to the service exposed by itself.
