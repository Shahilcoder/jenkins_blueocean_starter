## 0. Build the Docker Image

Before running the Docker Compose setup, you need to build the custom Jenkins BlueOcean image:

```bash
docker build -t myjenkins-blueocean:2.516.3-1 .
```

This builds a custom Jenkins image with:
- Jenkins 2.516.3 with JDK 21
- Docker CLI installed
- BlueOcean, Docker Workflow, and JSON Path API plugins pre-installed

## 1. What are docker-compose YAML files?

A `docker-compose.yml` file is basically a blueprint that describes **how to run and connect multiple containers together**.

Instead of starting containers one by one with long `docker run` commands, you define all the containers (called **services**) in one file, including:

* which images to use,
* what ports to expose,
* environment variables,
* networks,
* volumes.

Then you just run:

```bash
docker-compose up -d
```

And it spins up the whole environment in one go.

Think of it as **infrastructure as code for containers**.

## 2. Break down of `docker-compose.yml` line by line


```yaml
services:
```

**Top-level key**. Defines all the containers (services) you want to run.

### Service 1: `jenkins_docker`

```yaml
  jenkins_docker:
```

Defines a container named `jenkins_docker`. This will run Docker-in-Docker, so Jenkins can build and run Docker containers.

```yaml
    # container_name: jenkins-docker
```

Commented out. If enabled, it forces the container name to be `jenkins-docker`.
Without this, Docker auto-generates a name.

```yaml
    image: docker:dind
```

Uses the **Docker-in-Docker (dind)** image. This lets you run a Docker daemon inside the container, so Jenkins can build Docker images.

```yaml
    privileged: true
```

Gives the container extra permissions. Needed for running Docker inside Docker.

```yaml
    networks:
      jenkins:
        aliases:
          - docker
```

Connects this container to a custom network named `jenkins`.
It also creates an **alias** `docker`, so other containers can reach it at `docker:2376` instead of by IP.

```yaml
    volumes:
      - jenkins-docker-certs:/certs/client
      - jenkins-data:/var/jenkins_home
```

Mounts two **named volumes**:

* `jenkins-docker-certs` → stores Docker TLS certificates.
* `jenkins-data` → persists Jenkins data (so it survives container restarts).

```yaml
    environment:
      - DOCKER_TLS_CERTDIR=/certs
```

Tells Docker to store TLS certificates in `/certs`.

```yaml
    ports:
      - 2376:2376
```

Exposes Docker’s TLS port (2376) on the host machine, so other services (like Jenkins BlueOcean) can talk to it.

```yaml
    command: --storage-driver=overlay2
```

Overrides the container’s default command, telling Docker to use the `overlay2` storage driver (efficient layer handling for Docker images).

---

### Service 2: `jenkins_blueocean`

```yaml
  jenkins_blueocean:
```

Defines another container, running Jenkins with BlueOcean UI.

```yaml
    # container_name: jenkins-blueocean
```

Again, commented out, but could give it a fixed name.

```yaml
    image: myjenkins-blueocean:2.516.3-1
```

Uses a **custom Jenkins BlueOcean image** (built via given Dockerfile)  and tagged as `myjenkins-blueocean:2.516.3-1`.

```yaml
    restart: on-failure
```

Tells Docker to restart this container if it crashes.

```yaml
    depends_on:
      - jenkins_docker
```

Ensures `jenkins_docker` (Docker daemon) starts before this Jenkins container.

```yaml
    networks:
        - jenkins
```

Connects this container to the same `jenkins` network, so it can talk to the Docker daemon.

```yaml
    volumes:
      - jenkins-docker-certs:/certs/client
      - jenkins-data:/var/jenkins_home
```

Mounts the same volumes as `jenkins_docker`.
This way, both services share Jenkins home and Docker certs.

```yaml
    environment:
      - DOCKER_HOST=tcp://docker:2376
      - DOCKER_CERT_PATH=/certs/client
      - DOCKER_TLS_VERIFY=1
```

These vars tell Jenkins how to connect to the Docker daemon:

* `DOCKER_HOST` → points to the alias `docker` (from `jenkins_docker`) on port 2376.
* `DOCKER_CERT_PATH` → path to TLS certificates for authentication.
* `DOCKER_TLS_VERIFY=1` → ensures secure connection.

```yaml
    ports:
      - 8080:8080
      - 50000:50000
```

Maps ports:

* `8080` → Jenkins web UI.
* `50000` → Jenkins agent communication port.

---

### Networks

```yaml
networks:
  jenkins:
    driver: bridge
```

Defines a **custom bridge network** named `jenkins`.
Containers inside this network can talk to each other using names instead of IPs.

---

### Volumes

```yaml
volumes:
  jenkins-docker-certs:
  jenkins-data:
```

Creates **named volumes** for persistent storage.
Unlike `bind mounts`, these survive container deletion.

---

## In summary

This compose file sets up:

* **`jenkins_docker`** → A Docker daemon running in a container.
* **`jenkins_blueocean`** → Jenkins with BlueOcean UI, configured to use that Docker daemon.
* Shared volumes for persistence.
* A custom bridge network so containers can talk by name.
* Exposed ports for access from outside.

---

⚡ **Why is this useful?**
Because Jenkins can’t just build Docker images on its own. This setup lets Jenkins talk to a “real” Docker daemon (`jenkins_docker`) securely, without messing with your host’s Docker engine.

