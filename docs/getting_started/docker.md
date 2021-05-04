# HORNET in Docker

Hornet Docker images (amd64/x86_64 architecture) are available at [gohornet/hornet](https://hub.docker.com/r/gohornet/hornet) Docker hub.

## Requirements

1. A recent release of Docker enterprise or community edition. (Follow this [link](https://docs.docker.com/engine/install/) for install instructions).
2. git and curl
3. At least 1GB available RAM

### Clone Repository

Clone the repository

```sh
git clone https://github.com/gohornet/hornet && cd hornet
```

The rest of the document assumes you are executing commands from the root directory of the repository.

### Prepare

1. Edit the `config.json` for alternative ports if needed.

2. Edit `peering.json` to your neighbors addresses.

3. The Docker image runs under user with user id 65532 and group id 65532. To make sure there are no permission issues, create the directory for the database, e.g.:

   ```sh
   sudo mkdir mainnetdb && sudo chown 65532:65532 mainnetdb
   ```

4. The Docker image runs under user with user id 65532 and group id 65532. To make sure there are no permission issues, create the directory for the snapshots, e.g.:

   ```sh
   sudo mkdir -p snapshots/mainnet && sudo chown 65532:65532 snapshots -R
   ```

### Run

Pull the latest image from `gohornet/hornet` public Docker hub registry:

```bash
docker pull gohornet/hornet:latest
```

Best is to run on host network for better performance (otherwise you are going to have to publish ports, that is done via iptables NAT and is slower)

```sh
docker run --rm \
  -v $(pwd)/config.json:/app/config.json:ro \
  -v $(pwd)/peering.json:/app/peering.json \
  -v $(pwd)/profiles.json:/app/profiles.json \
  -v $(pwd)/mainnetdb:/app/mainnetdb \
  -v $(pwd)/snapshots/mainnet:/app/snapshots/mainnet \
  --name hornet\
  --net=host \
  hornet:latest
```

Use CTRL-c to gracefully end the process.

* `$(pwd)`: stands for the current directory
* `-d`: instructs Docker to run the container instance in a detached mode (daemon).
* `--restart always`: instructs Docker the given container is restarted after Docker reboot
* `--name hornet`: a name of the running container instance. You can refer to the given container by this name
* `--net=host`: instructs Docker to use directly network on host (so the network is not isolated). The best is to run on host network for better performance. It also means it is not necessary to specify any ports. Ports that are opened by container are opened directly on the host
* `-v $(pwd)/config.json:/app/config.json:ro`: it maps the local `config.json` file into the container in `readonly` mode
* `-v $(pwd)/snapshots/mainnet:/app/snapshots/mainnet`: it maps the local `snapshots` directory into the container
* `-v $(pwd)/mainnetdb:/app/mainnetdb`: it maps the local `mainnetdb` directory into the container
* all mentioned directories are mapped to the given container and so the Hornet in container persists the data directly to those directories

### Create username and password for the Hornet dashboard

If you use the Hornet dashboard you need to create a secure password. Start your Hornet container and run the following command:

```sh
docker exec -it hornet /app/hornet tool pwdhash

Re-enter your password:
Success!
Your hash: a24c5f65afd0ff0a37af47ff8c02a60971f097b1fd8f7e3739105cca9bf294c4
Your salt: 808fe4e5cd22c8bc7d8b0d34f2d7b96325a9f0d6e633f3241a1be1ff537e956c
```

Edit `config.json` and customize the "dashboard" section to your needs.

```sh
  "dashboard": {
    "bindAddress": "0.0.0.0:8081",
    "auth": {
      "sessionTimeout": "72h",
      "username": "admin",
      "passwordHash": "a24c5f65afd0ff0a37af47ff8c02a60971f097b1fd8f7e3739105cca9bf294c4",
      "passwordSalt": "808fe4e5cd22c8bc7d8b0d34f2d7b96325a9f0d6e633f3241a1be1ff537e956c"
    }
  },
```

### Build your own hornet image

If not running via docker-compose, build the image manually:

```sh
docker build -f docker/Dockerfile -t hornet:latest .
```

Or pull it from Docker hub (only available for amd64/x86_64):

```sh
docker pull gohornet/hornet:latest && docker tag gohornet/hornet:latest hornet:latest
```

### Build your own image using Docker Compose

Note: Follow this step only if you want to run HORNET via docker-compose.

If you are using an architecture other than amd64/x86_64 edit the `docker-compose.yml` file and set the correct architecture where noted.

The following command will build the image and run HORNET:

```sh
docker-compose up
```

CTRL-c to stop.

Add `-d` to run detached, and to stop:

```sh
docker-compose down -t 1200
```

### Using your own Docker Compose file for running Hornet

Docker-compose is a tool on top of the Docker engine that enables you (among other features) to define Docker parameters in a structured way via `yaml` file. Then, with a single `docker-compose` command, you create and start containers based on your configuration.

Create `docker-compose.yml` file among the other files created above:

```plaintext
.
├── config.json
├── peering.json
├── docker-compose.yml      <NEWLY ADDED FILE>
├── mainnetdb
└── snapshots
    └── mainnet
```

```plaintext
version: '3'
services:
  hornet:
    container_name: hornet
    image: gohornet/hornet:latest
    network_mode: host
    restart: always
    cap_drop:
      - ALL
    volumes:
      - ./config.json:/app/config.json:ro
      - ./peering.json:/app/peering.json
      - ./snapshots/mainnet:/app/snapshots/mainnet
      - ./mainnetdb:/app/mainnetdb
```

Then run `docker-compose up` in the current directory:

```bash
docker-compose up
```

* it reads parameters from the given `docker-compose.yml` and fires up the container named `hornet` based on them
* `docker-compose up` automatically runs containers in detached mode (as daemon)

Restarting and managing containers that were created by `docker-compose` can be done in the same fashion as mentioned above (`docker log`, `docker restart`, etc.).

See more details on how to configure Hornet under the [post installation](../post_installation/post_installation.md) chapter.

### Managing node

## Displaying log output

```bash
docker log -f hornet
```

* `-f`: instructs Docker to continue displaying the log to `stdout` until CTRL+C is pressed

## Restarting Hornet

```bash
docker restart -t 200 hornet
```

## Stopping Hornet

```bash
docker stop -t 200 hornet
```

* `-t 200`: instructs Docker to wait for a grace period before shutting down

> Please note: Hornet uses an in-memory cache and so it is necessary to provide a grace period while shutting it down (at least 200 seconds) in order to save all data to the underlying persistent storage.

## Removing container

```bash
docker container rm hornet
```
