# Docker - Answers

## 1-1 Why is it better to use `-e` for environment variables rather than putting them in the Dockerfile?

Hardcoding credentials (passwords, usernames) directly in the Dockerfile is a security risk: the file is usually committed to a version control system (e.g. Git), making secrets visible to anyone with repository access. It also gets baked into the image layers, which can be inspected with `docker history` or `docker inspect`.

Using `-e` at runtime keeps secrets out of the image and the source code. It also makes the image reusable across environments (dev, staging, prod) with different credentials, without having to rebuild.

---

## 1-2 Why do we need a volume attached to our postgres container?

Containers are ephemeral: when a container is removed, all data stored inside it is lost. A database must persist data durably across restarts and container recreations. Attaching a volume maps a directory on the host machine to the container's data directory (`/var/lib/postgresql/data`), ensuring data survives even if the container is destroyed and recreated.

---

## 1-3 Database container essentials

**Dockerfile:**
```dockerfile
FROM postgres:17.2-alpine

COPY ./sql-scripts/ /docker-entrypoint-initdb.d/
```

Credentials are passed at runtime via `-e` flags (see below).

**Commands:**
```bash
# Create the network
docker network create app-network

# Build the image
docker build -t my-database ./database

# Run the container
docker run -d \
  --name my-postgres \
  --net=app-network \
  -e POSTGRES_DB=db \
  -e POSTGRES_USER=usr \
  -e POSTGRES_PASSWORD=pwd \
  -v /my/own/datadir:/var/lib/postgresql/data \
  my-database

# Run Adminer (for DB inspection)
docker run \
  -p "8090:8080" \
  --net=app-network \
  --name=adminer \
  -d \
  adminer
```

---

## 1-4 Why do we need a multistage build? Explain each step of the Dockerfile.

A multistage build separates the **build environment** from the **runtime environment**. The first stage uses a full JDK + Maven image to compile and package the application. The second stage uses a lightweight JRE-only image and copies just the resulting `.jar`. This keeps the final image small (no build tools, no source code), reducing both image size and attack surface.

**Step-by-step explanation:**

```dockerfile
# --- Stage 1: Build ---
FROM eclipse-temurin:21-jdk-alpine AS myapp-build
# Full JDK image, capable of compiling Java code. Named "myapp-build" for reference later.

ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
# Defines and sets the working directory inside the container.

RUN apk add --no-cache maven
# Installs Maven using Alpine's package manager.

COPY pom.xml .
COPY src ./src
# Copies the project descriptor and source code into the container.

RUN mvn package -DskipTests
# Compiles and packages the application into a .jar, skipping tests for speed.

# --- Stage 2: Run ---
FROM eclipse-temurin:21-jre-alpine
# Lightweight JRE-only image. No compiler, no build tools.

ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME

COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
# Copies only the built .jar from Stage 1 into this clean runtime image.

ENTRYPOINT ["java", "-jar", "myapp.jar"]
# Starts the application when the container runs.
```

---

## 1-5 Why do we need a reverse proxy?

A reverse proxy (here: Apache httpd) sits in front of the backend and acts as a single entry point for all incoming requests. It provides several benefits:

- **Abstraction**: clients only talk to one address; the backend's internal host/port stays hidden.
- **Security**: the backend is not directly exposed to the outside world.
- **Flexibility**: the proxy can handle SSL termination, load balancing, caching, or serve static frontend files — all without changing the backend.
- **Routing**: multiple services can be exposed through a single port with path-based or host-based routing.

---

## 1-6 Why is docker-compose so important?

Manually starting, stopping, networking, and rebuilding multiple containers is error-prone and tedious. `docker-compose` lets you declare your entire multi-container application in a single `docker-compose.yml` file. It handles:

- Bringing all services up or down with a single command.
- Automatic network creation between services.
- Dependency ordering (`depends_on`).
- Volume and environment variable management.
- Reproducibility: anyone on the team can run the exact same stack with one command.

---

## 1-7 Most important docker-compose commands

```bash
# Start all services (build images if needed, run in background)
docker compose up -d

# Stop and remove all containers, networks
docker compose down

# Rebuild images before starting
docker compose up -d --build

# View logs for all services (or a specific one)
docker compose logs -f
docker compose logs -f backend

# List running services
docker compose ps

# Run a one-off command inside a service container
docker compose exec backend sh
```

---

## 1-8 docker-compose file

```yaml
services:
  backend:
    build: ./backend/simple-api
    networks:
      - app-network
    depends_on:
      - database
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://database:5432/db
      - SPRING_DATASOURCE_USERNAME=usr
      - SPRING_DATASOURCE_PASSWORD=pwd
    restart: unless-stopped

  database:
    build: ./database
    networks:
      - app-network
    environment:
      - POSTGRES_DB=db
      - POSTGRES_USER=usr
      - POSTGRES_PASSWORD=pwd
    volumes:
      - db-data:/var/lib/postgresql/data
    restart: unless-stopped

  httpd:
    build: ./httpd
    ports:
      - "80:80"
    networks:
      - app-network
    depends_on:
      - backend
    restart: unless-stopped

networks:
  app-network:

volumes:
  db-data:
```

Key points:
- Only `httpd` exposes a port to the host (`80:80`). The backend and database are only reachable internally via `app-network`.
- Credentials are defined as environment variables, not baked into images.
- A named volume `db-data` ensures database persistence.
- `depends_on` controls startup order.
- `restart: unless-stopped` ensures services recover automatically after a crash.

---

## 1-9 Publication commands and published images

```bash
# Login to Docker Hub
docker login

# Tag images with your username and a version
docker tag my-database USERNAME/my-database:1.0
docker tag my-backend USERNAME/my-backend:1.0
docker tag my-httpd USERNAME/my-httpd:1.0

# Push to Docker Hub
docker push USERNAME/my-database:1.0
docker push USERNAME/my-backend:1.0
docker push USERNAME/my-httpd:1.0
```

Published images are then available at:
- `docker.io/USERNAME/my-database:1.0`
- `docker.io/USERNAME/my-backend:1.0`
- `docker.io/USERNAME/my-httpd:1.0`

---

## 1-10 Why do we put our images into an online repo?

Publishing images to a registry (like Docker Hub) makes them available to other team members, CI/CD pipelines, and remote servers without needing to rebuild from source. It enables:

- **Collaboration**: teammates can pull and run the exact same image without having the source code.
- **Deployment**: production servers can pull versioned images directly.
- **Versioning**: tags allow rolling back to a previous working image if needed.
- **Reproducibility**: the same image runs identically in every environment.