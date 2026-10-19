# Overview
A quick reference for commonly used Docker commands, their important flags, and example usage.

---
## 1. `docker run`

Create and start a new container from an image.

**Common flags**

|Flag|Description|
|---|---|
|`-d`|Run in detached mode (background).|
|`-p HOST:CONTAINER`|Map a host port to a container port.|
|`-v HOST:CONTAINER`|Mount a host path or volume into the container.|
|`--name NAME`|Assign a name to the container.|
|`--rm`|Automatically remove the container after it stops.|
|`-e KEY=VALUE`|Set an environment variable.|
|`--network NAME`|Connect the container to a Docker network.|
|`--restart POLICY`|Set a restart policy: `no`, `always`, `unless-stopped`, or `on-failure`.|

**Example**

```
docker run -d --name nginx -p 8080:80 --restart unless-stopped nginx
```

---
## 2. `docker build`

Build a Docker image from a Dockerfile.

**Common flags**

|Flag|Description|
|---|---|
|`-t NAME`|Assign a name and optional tag to the image.|
|`-f FILE`|Specify the Dockerfile path.|
|`--no-cache`|Build without using cached layers.|
|`--pull`|Attempt to pull newer versions of base images.|

**Example**

```
docker build -t myapp:1.0 .
```

---
## 3. `docker pull`

Download an image from a container registry.

**Usage**

```
docker pull nginx:latest
```

Specify a tag to download a particular version. If omitted, Docker uses the `latest` tag by default.

---
## 4. `docker push`

Upload an image to a container registry.

**Example**

```
docker push username/myapp:1.0
```

Authenticate with the registry first if required.

---
## 5. `docker ps`

List containers.

**Common flags**

|Flag|Description|
|---|---|
|`-a`|Show all containers, including stopped ones.|
|`-q`|Display only container IDs.|
|`--filter KEY=VALUE`|Filter containers by a condition.|

**Examples**

```
docker ps
docker ps -a
docker ps -q
docker ps --filter "status=exited"
```

---
## 6. `docker images`

List locally available images.

**Common flags**

|Flag|Description|
|---|---|
|`-a`|Show all images, including intermediate images.|
|`-q`|Display only image IDs.|

**Example**

```
docker images
```

---
## 7. `docker exec`

Execute a command inside a running container.

**Common flags**

|Flag|Description|
|---|---|
|`-i`|Keep standard input open.|
|`-t`|Allocate a pseudo-terminal.|
|`-u USER`|Run the command as a specific user.|

**Examples**

```
docker exec -it nginx bash
docker exec -u root -it nginx sh
```

Use `sh` if the container does not include Bash.

---
## 8. `docker logs`

View a container's logs.

**Common flags**

|Flag|Description|
|---|---|
|`-f`|Follow logs in real time.|
|`--tail N`|Display the last N lines.|
|`--since TIME`|Show logs since a specified time.|
|`-t`|Include timestamps.|

**Examples**

```
docker logs nginx
docker logs -f nginx
docker logs --tail 100 nginx
```

---
## 9. `docker stop`

Gracefully stop one or more running containers.

```
docker stop nginx
```

Docker sends a termination signal and waits for the configured timeout before forcing termination.

---
## 10. `docker kill`

Immediately send a signal to a running container. By default, it sends `SIGKILL`.

```
docker kill nginx
```

Use this when a container needs to be stopped immediately rather than gracefully.

---
## 11. `docker rm`

Remove one or more containers.

**Common flags**

|Flag|Description|
|---|---|
|`-f`|Force-remove a running container.|
|`-v`|Remove associated anonymous volumes.|

**Examples**

```
docker rm nginx
docker rm -f nginx
```

Removing a container does not automatically remove named volumes.

---
## 12. `docker rmi`

Remove one or more images.

**Common flags**

|Flag|Description|
|---|---|
|`-f`|Force removal of an image.|

**Example**

```
docker rmi nginx:latest
```

An image may need to be unreferenced by containers before it can be removed.

---
## 13. `docker inspect`

Display detailed information about Docker objects, including containers, images, networks, and volumes.

```
docker inspect nginx
docker inspect --format '{{.State.Status}}' nginx
```

The default output is JSON containing metadata and configuration details.

---
## 14. `docker network create`

Create a Docker network.

**Common flags**

|Flag|Description|
|---|---|
|`--driver DRIVER`|Specify the network driver, such as `bridge` or `overlay`.|
|`--subnet CIDR`|Define the network's IP address range.|

**Example**

```
docker network create --driver bridge media_net
```

---
## 15. `docker volume create`

Create a persistent Docker volume.

```
docker volume create media_config
```

**Note:** Docker manages named volumes, which can persist data independently of container lifecycles.

---
## 16. `docker compose up`

Create and start services defined in a Compose file.

**Common flags**

|Flag|Description|
|---|---|
|`-d`|Run services in the background.|
|`--build`|Build images before starting services.|
|`--force-recreate`|Recreate containers even if their configuration has not changed.|
|`--remove-orphans`|Remove containers for services no longer defined in the Compose file.|

**Examples**

```
docker compose up -d
docker compose up -d --build
docker compose up -d --force-recreate
```

---
## 17. `docker compose down`

Stop and remove containers and networks created by a Compose project.

**Common flags**

|Flag|Description|
|---|---|
|`-v`|Also remove declared named and anonymous volumes associated with the project.|
|`--rmi all`|Remove all images used by the services.|
|`--remove-orphans`|Remove containers for services no longer defined in the Compose file.|

**Examples**

```
docker compose down
docker compose down --remove-orphans
```

**Warning:** Use `-v` carefully because removing volumes can permanently delete persistent application data.

---
## 18. `docker tag`

Assign another name or version tag to an image.

```
docker tag myapp:1.0 username/myapp:1.0
```

Tagging does not rebuild the image. It creates another reference to the same image.

---
## 19. `docker login`

Authenticate with a container registry.

```
docker login
docker login ghcr.io
```

Use this when you need to pull private images or push images to a registry.

## 20. `docker logout`

Log out of a container registry.

```
docker logout
docker logout ghcr.io
```

This removes the locally stored authentication credentials for the specified registry.

---
