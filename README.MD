# AntreJeu — Deployment Repository

This repository assembles the AntreJeu application for deployment. It contains the Docker Compose configuration, Nginx configuration, and Git submodules referencing the frontend and backend repositories.

## Architecture

AntreJeu is organized into three independent Git repositories:

| Repository | Responsibility |
|---|---|
| `AntreJeu-front` | React frontend, Vite build, frontend tests and quality checks |
| `AntreJeu-back` | Express backend and API |
| `AntreJeu` | Deployment configuration, Docker Compose, Nginx and submodule references |

The deployment repository pins specific commits of the frontend and backend. This makes it possible to identify exactly which versions are deployed together.

### Docker architecture

```text
Internet
   |
   v
Host port 8080
   |
   v
Nginx reverse proxy (proxy container)
   |
   +----> Frontend container (Nginx + React build)
   |
   +----> Backend container (Express API)
                 |
                 +----> Database container (future: MySQL)
```

The current setup exposes the reverse proxy on host port `8080`. The frontend is reachable through the proxy, rather than directly through a published host port.

Docker Compose creates a private bridge network for communication between containers. Services can address one another by their Compose service names.

The two Nginx configurations have different purposes:

- `docker/nginx/proxy.conf`: the public entry point. It forwards incoming requests to the appropriate service.
- `docker/nginx/frontend.conf`: the Nginx server inside the frontend container. It serves the compiled React application and falls back to `index.html` for client-side routes.


## Prerequisites

- Git
- Docker Desktop for local development, or Docker Engine and the Compose plugin on Linux
- Access to the frontend and backend Git repositories

## Getting started

### Clone the deployment repository

Clone with its submodules:

```bash
git clone --recurse-submodules https://github.com/Fafacraft/AntreJeu.git
cd AntreJeu
```

If the repository has already been cloned without its submodules:

```bash
git submodule update --init --recursive
```

### Configure environment variables

Create a local `.env` file in the root of this repository, alongside `compose.yaml`.

Keep `.env` out of Git. Commit an `.env.example` containing placeholder values so other developers know which variables are required.


### Start the application

Build the images and start the containers:

```bash
docker compose up --build -d
```

Open the application at:

http://localhost:8080

Useful commands:

```bash
# Validate the Compose configuration
docker compose config

# List running containers
docker compose ps

# Follow logs
docker compose logs -f

# Follow proxy logs
docker compose logs -f proxy

# Rebuild the frontend without using cached build layers
docker compose build --no-cache frontend

# Recreate the services
docker compose up -d --force-recreate

# Stop the application and remove containers and network
docker compose down
```

`docker compose down` does not delete named volumes by default. If persistent database storage is added later, take care not to remove it unintentionally.

## Git workflow: development, staging and production

The project uses three branches across the application and deployment repositories:

- `feature/*`: development work.
- `staging`: integration and validation.
- `main`: production-ready version.

The same promotion process applies to the frontend, backend, and deployment repository.

### 1. Develop a feature

Create a feature branch in the relevant application repository:

```bash
git switch staging
git pull origin staging
git switch -c feature/my-feature
```

Make changes, run the quality checks, commit, and push:

```bash
git add .
git commit -m "Implement my feature"
git push -u origin feature/my-feature
```

Open a pull request from `feature/my-feature` into `staging`.

Only merge after the relevant checks pass and the change has been reviewed.

### 2. Integrate into staging

Merge the feature into `staging` through a pull request.

Repeat this process independently for the frontend and/or backend with features until a version is ready.

### 2.1 Testing features and staging

The deployment repository's can be used to test other submodules branch (staging or feature) temporarily by updating its submodules locally with :

```bash
git -C [submodule] fetch origin [branch]
git -C [submodule] checkout origin/[branch]
```

These commands check out the latest fetched `[branch]` commit in each submodule. They do not merge application branches or change the configured submodule tracking branches.

Build and test the integrated application:

```bash
docker compose up --build -d
docker compose ps
docker compose logs -f
```

### 3. Promote a validated release to production

Once the staging version has been validated, promote the application repositories to `main` through pull requests from `staging` to `main`.

Do this independently in the frontend and/or backend repositories.

Next, update the deployment repository's `main` branch by fetching the last `main` commits of the submodules

```bash
git submodule update --remote
```

Commit the updated references in the deployment repository's release branch, validate the resulting Compose configuration and images, and merge the deployment release into the deployment repository's `main`.


### Important: submodules pin commits

A submodule is not a live link that automatically follows another repository.

The deployment repository stores a specific commit ID for each submodule. Updating a submodule changes the commit reference in the parent repository, which must then be committed.

For example:

```bash
# Fetch a particular branch
git -C frontend fetch origin staging

# Check out that branch's latest fetched commit
git -C frontend checkout origin/staging
```

To update submodules using their configured remote-tracking branches (should be `main`):

```bash
git submodule update --remote
```


## Environment variables and secrets

### Build-time frontend configuration

Vite embeds variables prefixed with `VITE_` into the JavaScript bundle at build time.

The Compose configuration passes the public Sentry DSN to the frontend Docker build:

```yaml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      args:
        VITE_SENTRY_DSN: ${VITE_SENTRY_DSN}
```

The frontend Dockerfile must declare and expose the argument before running the Vite build:

```dockerfile
ARG VITE_SENTRY_DSN
ENV VITE_SENTRY_DSN=${VITE_SENTRY_DSN}

RUN npm run build
```

Changing the DSN requires rebuilding the frontend image:

```bash
docker compose build frontend
docker compose up -d frontend
```


## Credits
Fabien ARTHUR