# enshrouded-server

[![Static Badge](https://img.shields.io/badge/DockerHub-blue)](https://hub.docker.com/r/sknnr/enshrouded-dedicated-server) ![Docker Pulls](https://img.shields.io/docker/pulls/sknnr/enshrouded-dedicated-server) [![Static Badge](https://img.shields.io/badge/GitHub-green)](https://github.com/jsknnr/enshrouded-server) ![GitHub Repo stars](https://img.shields.io/github/stars/jsknnr/enshrouded-server)

Run Enshrouded dedicated server in a container. Optionally includes helm chart for running in Kubernetes.

**Disclaimer:** This is not an official image. No support, implied or otherwise is offered to any end user by the author or anyone else. Feel free to do what you please with the contents of this repo.

## Usage

The processes within the container do **NOT** run as root. Everything runs as the user steam (gid:10000/uid:10000 by default). If you exec into the container, you will drop into `/home/steam` as the steam user. Enshrouded will be installed to `/home/steam/enshrouded`. Any persistent volumes should be mounted to `/home/steam/enshrouded/savegame` and be owned by 10000:10000.

If you absolutely require to run the process in the container as a gid/uid other than 10000, you can build your own image based on my dockerfile. Instructions are covered [Here](https://github.com/jsknnr/enshrouded-server/issues/51)

### Proton and Wine based images

The `latest` tag is now based on the Proton build instead of Wine. This should be seamless. Outside of `latest`, there is `wine-$realease_version` and `proton-$release_version` with `$release_version` being the version of the release from GitHub. I am no longer updating the Wine version of this image.

### Ports

| Port       | Protocol | Default |
| ---------- | -------- | ------- |
| Query Port | UDP      | 15637   |

### Environment Variables

| Name            | Description                                                             | Default                  | Required |
| --------------- | ----------------------------------------------------------------------- | ------------------------ | -------- |
| SERVER_NAME     | Name for the Server                                                     | Enshrouded Containerized | False    |
| SERVER_PASSWORD | Password for the server                                                 | None                     | False    |
| PORT            | Port for steam query of server                                          | 15637                    | False    |
| SERVER_SLOTS    | Number of slots for connections (Max 16)                                | 16                       | False    |
| SERVER_IP       | IP address for server to listen on                                      | 0.0.0.0                  | False    |
| EXTERNAL_CONFIG | If you would rather manually supply a config file, set this to true (1) | 0                        | False    |

**Note:** SERVER_IP is ignored if using Helm because that isn't how Kubernetes works.

### Docker

To run the container in Docker, run the following command:

```bash
docker volume create enshrouded-persistent-data
docker run \
  --detach \
  --name enshrouded-server \
  --mount type=volume,source=enshrouded-persistent-data,target=/home/steam/enshrouded/savegame \
  --publish 15637:15637/udp \
  --env=SERVER_NAME='Enshrouded Containerized Server' \
  --env=SERVER_SLOTS=16 \
  --env=SERVER_PASSWORD='ChangeThisPlease' \
  --env=PORT=15637 \
  sknnr/enshrouded-dedicated-server:latest
```

### Docker Compose

To use Docker Compose, either clone this repo or copy the `compose.yaml` file out of the `container` directory to your local machine. Edit the compose file to change the environment variables to the values you desire and then save the changes. Once you have made your changes, from the same directory that contains the compose and the env files, simply run:

```bash
docker-compose up -d
```

To bring the container down:

```bash
docker-compose down
```

compose.yaml file:

```yaml
version: "3"
services:
  enshrouded:
    image: sknnr/enshrouded-dedicated-server:latest
    ports:
      - "15637:15637/udp"
    environment:
      - SERVER_NAME=Enshrouded Containerized
      - SERVER_PASSWORD=PleaseChangeMe
      - PORT=15637
      - SERVER_SLOTS=16
      - SERVER_IP=0.0.0.0
    volumes:
      - enshrouded-persistent-data:/home/steam/enshrouded/savegame

volumes:
  enshrouded-persistent-data:
```

### Podman

To run the container in Podman, run the following command:

```bash
podman volume create enshrouded-persistent-data
podman run \
  --detach \
  --name enshrouded-server \
  --mount type=volume,source=enshrouded-persistent-data,target=/home/steam/enshrouded/savegame \
  --publish 15637:15637/udp \
  --env=SERVER_NAME='Enshrouded Containerized Server' \
  --env=SERVER_SLOTS=16 \
  --env=SERVER_PASSWORD='ChangeThisPlease' \
  --env=PORT=15637 \
  docker.io/sknnr/enshrouded-dedicated-server:latest
```

### Quadlet

To run the container with Podman's new quadlet subsystem, make a file under (when running as root) /etc/containers/systemd/enshrouded.container containing:

```text
[Unit]
Description=Enshrouded Game Server

[Container]
Image=docker.io/sknnr/enshrouded-dedicated-server:latest
Volume=enshrouded-persistent-data:/home/steam/enshrouded/savegame
PublishPort=15637:15637/udp
ContainerName=enshrouded-server
Environment=SERVER_NAME="Enshrouded Containerized Server"
Environment=SERVER_PASSWORD="ChangeThisPlease"
Environment=PORT=15637
Environment=SERVER_SLOTS=16

[Service]
# Restart service when sleep finishes
Restart=always
# Extend Timeout to allow time to pull the image
TimeoutStartSec=900

[Install]
# Start by default on boot
WantedBy=multi-user.target default.target
```

### Kubernetes

I've built a Helm chart and have included it in the `helm` directory within this repo. Modify the `values.yaml` file to your liking and install the chart into your cluster. Be sure to create and specify a namespace as I did not include a template for provisioning a namespace.

The chart in this repo is also hosted in my helm-charts repository [here](https://jsknnr.github.io/helm-charts)

To install this chart from my helm-charts repository:

```bash
helm repo add jsknnr https://jsknnr.github.io/helm-charts
helm repo update
```

To install the chart from the repo:

```bash
helm install enshrouded jsknnr/enshrouded-dedicated-server --values myvalues.yaml
# Where myvalues.yaml is your copy of the Values.yaml file with the settings that you want
```

## Troubleshooting

### Connectivity

If you are having issues connecting to the server once the container is deployed, I promise the issue is not with this image. You need to make sure that the ports 15636 and 15637 (or whichever ones you decide to use) are open on your router as well as the container host where this container image is running. You will also have to port-forward the game-port and query-port from your router to the private IP address of the container host where this image is running. After this has been done correctly and you are still experiencing issues, your internet service provider (ISP) may be blocking the ports and you should contact them to troubleshoot.

For additional help, refer to this closed issue where some folks were able to debug their issues. It may be of help. <br>
https://github.com/jsknnr/enshrouded-server/issues/16

### Storage

I recommend having Docker or Podman manage the volume that gets mounted into the container. However, if you absolutely must bind mount a directory into the container you need to make sure that on your container host the directory you are bind mounting is owned by 10000:10000 by default (`chown -R 10000:10000 /path/to/directory`). If the ownership of the directory is not correct the container will not start as the server will be unable to persist the savegame.
