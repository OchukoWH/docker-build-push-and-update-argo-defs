# VProfile - Docker Containerized Application

A comprehensive demonstration of Docker containerization expertise, featuring a multi-tier Java web application with microservices architecture.

## 🐳 Docker Expertise Showcase

This project demonstrates advanced Docker concepts including:
- **Multi-container orchestration** with Docker Compose
- **Multi-stage builds** for optimized images
- **Custom Dockerfile optimizations** for production readiness
- **Service discovery** and inter-container communication
- **Persistent data management** with Docker volumes
- **Load balancing** with Nginx reverse proxy
- **Database containerization** with initialization scripts
- **CI/CD pipeline** with GitHub Actions for automated builds and deployments
- **Multi-environment deployments** (dev/staging/prod) with Docker Hub integration
- **Image versioning** with intelligent tagging strategies

## 🏗️ Architecture Overview

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Nginx Proxy   │    │   Tomcat App    │    │   MySQL DB      │
│   (Port 80)     │◄──►│   (Port 8080)   │◄──►│   (Port 3306)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐    ┌─────────────────┐
                       │   Memcached     │    │   RabbitMQ      │
                       │   (Port 11211)  │    │   (Port 5672)   │
                       └─────────────────┘    └─────────────────┘
```

### Prerequisites
- Docker Engine 20.10+
- Docker Compose 2.0+
- Git

### Running the Application

1. **Clone the repository**
   ```bash
   git clone https://github.com/CK-codemax/vprofile-docker.git
   cd vprofile-docker
   ```

2. **Build and start all services**
   ```bash
   docker-compose up --build
   ```

3. **Access the application**
   - **Web Application**: http://localhost:80 or http://0.0.0.0:80 (default port: **80**)
   - **Tomcat Direct**: http://localhost:8080 (default port: **8080**)
   - **Database**: localhost:3306
   - **Memcached**: localhost:11211
   - **RabbitMQ**: localhost:5672

4. **Login to the application**
   - **Username**: `admin_vp`
   - **Password**: `admin_vp`
   - **(Webapp default login)**


## 🛠️ Development Workflow

### Building Individual Services
```bash
# Build application container
docker build -f Docker-files/app/Dockerfile -t vprofile-app .

# Build database container
docker build -f Docker-files/db/Dockerfile -t vprofile-db ./Docker-files/db

# Build web proxy
docker build -f Docker-files/web/Dockerfile -t vprofile-web ./Docker-files/web
```

### Development Commands
```bash
# Start services in background
docker-compose up -d

# View logs
docker-compose logs -f

# Stop all services
docker-compose down

# Rebuild specific service
docker-compose up --build vproapp

# Access container shell
docker exec -it vproapp /bin/bash
```


## 🔄 CI/CD Pipeline with GitHub Actions

This project includes a **complete CI/CD (Continuous Integration/Continuous Deployment) pipeline** that automatically builds and pushes Docker images to Docker Hub. The pipeline supports **three distinct environments** (development, staging, and production) with intelligent tagging to preserve image history.

### What is CI/CD?

**CI/CD** automates the software delivery process:
- **Continuous Integration (CI)**: Automatically builds and tests code when changes are pushed
- **Continuous Deployment (CD)**: Automatically deploys builds to Docker Hub for distribution

**How it works in this project:**
1. Developer pushes code to a Git branch
2. GitHub Actions workflow automatically triggers
3. Docker images are built from the Dockerfiles
4. Images are tagged and pushed to Docker Hub
5. Images can be pulled and deployed anywhere Docker is available

### Multi-Environment Strategy

The CI/CD pipeline is configured with **three separate environments**, each with its own branch and Docker Hub repositories:

| Environment | Branch Trigger | Docker Hub Repositories | Purpose |
|------------|----------------|------------------------|---------|
| **Development** | `dev` | `{username}/vprofiledb2`, `{username}/vprofileapp2`, `{username}/vprofileweb2` | Early testing and development |
| **Staging** | `staging` | Same repositories with `staging-*` tags | Pre-production testing |
| **Production** | `main`, `master`, or `docker-hub` | Same repositories with `prod-*` tags | Live production deployments |

### Image Tagging Strategy

Every push creates **two tags** for each image to preserve history:

1. **`{env}-latest`** - Points to the most recent build (overwrites previous `latest`)
   - Example: `dev-latest`, `staging-latest`, `prod-latest`

2. **`{env}-{commit-sha}`** - Unique tag based on Git commit SHA (preserved forever)
   - Example: `dev-a1b2c3d`, `staging-a1b2c3d`, `prod-a1b2c3d`

**Why two tags?**
- `{env}-latest` provides an easy way to always get the newest build
- `{env}-{commit-sha}` preserves all previous images for rollback or version tracking
- No images are ever lost or overwritten (unless you manually delete them)

### Docker Hub Repositories

All environments use the **same three repositories** on Docker Hub, differentiated by tags:

#### Repository 1: Database Image
- **Repository**: `{your-dockerhub-username}/vprofiledb2`
- **Available Tags**:
  - `dev-latest`, `dev-{sha}` (Development)
  - `staging-latest`, `staging-{sha}` (Staging)
  - `prod-latest`, `prod-{sha}` (Production)

#### Repository 2: Application Image
- **Repository**: `{your-dockerhub-username}/vprofileapp2`
- **Available Tags**: Same pattern as above

#### Repository 3: Web/NGINX Image
- **Repository**: `{your-dockerhub-username}/vprofileweb2`
- **Available Tags**: Same pattern as above

### Setup Instructions

#### Step 1: Create Docker Hub Access Token

1. Log in to [Docker Hub](https://hub.docker.com/)
2. Go to **Account Settings** → **Security** → **New Access Token**
3. Create a token with **read/write** permissions
4. **Copy and save the token** (you won't be able to see it again!)

#### Step 2: Configure GitHub Secrets

1. Go to your GitHub repository
2. Navigate to **Settings** → **Secrets and variables** → **Actions**
3. Click **"New repository secret"**
4. Add the following secrets:
   - **Name**: `DOCKERHUB_USERNAME`
     - **Value**: Your Docker Hub username
   - **Name**: `DOCKERHUB_TOKEN`
     - **Value**: The access token you created in Step 1

### How the Workflow Works

The workflow file (`.github/workflows/docker-image.yaml`) automatically:

1. **Detects the branch** you pushed to
2. **Selects the appropriate environment** (dev/staging/prod)
3. **Extracts the commit SHA** for tagging
4. **Sets up Docker Buildx** for optimized builds
5. **Authenticates** with Docker Hub using your secrets
6. **Builds all three images** in parallel:
   - Database image (`vprofiledb2`)
   - Application image (`vprofileapp2`)
   - Web/NGINX image (`vprofileweb2`)
7. **Tags each image** with both `{env}-latest` and `{env}-{sha}`
8. **Pushes to Docker Hub** for distribution

### Using Published Images

#### Pull Images from Docker Hub

```bash
# Development images
docker pull {your-username}/vprofiledb2:dev-latest
docker pull {your-username}/vprofileapp2:dev-latest
docker pull {your-username}/vprofileweb2:dev-latest

# Staging images
docker pull {your-username}/vprofiledb2:staging-latest
docker pull {your-username}/vprofileapp2:staging-latest
docker pull {your-username}/vprofileweb2:staging-latest

# Production images
docker pull {your-username}/vprofiledb2:prod-latest
docker pull {your-username}/vprofileapp2:prod-latest
docker pull {your-username}/vprofileweb2:prod-latest

# Pull specific version by commit SHA
docker pull {your-username}/vprofileapp2:prod-a1b2c3d
```

#### Update docker-compose.yml to Use Docker Hub Images

```yaml
services:
  vprodb:
    image: {your-username}/vprofiledb2:prod-latest  # or dev-latest, staging-latest
    container_name: vprodb
    # ... rest of config

  vproapp:
    image: {your-username}/vprofileapp2:prod-latest
    container_name: vproapp
    # ... rest of config

  vproweb:
    image: {your-username}/vprofileweb2:prod-latest
    container_name: vproweb
    # ... rest of config
```

#### Example: Deploy Specific Version

If you need to rollback to a previous version, you can use the commit SHA tag:

```yaml
vproapp:
  image: {your-username}/vprofileapp2:prod-a1b2c3d  # Specific commit
```

### Workflow Features

✅ **Automated builds** - No manual intervention needed  
✅ **Multi-environment support** - Separate dev, staging, and prod pipelines  
✅ **Image versioning** - Every build is tagged and preserved  
✅ **Parallel builds** - All three images build simultaneously for speed  
✅ **Secure credentials** - Uses GitHub Secrets for Docker Hub authentication  
✅ **Rollback capability** - Access any previous version via commit SHA tags  
✅ **Latest tags** - Easy access to the most recent build per environment  
✅ **Multi-stage optimization** - Smaller, optimized Docker images



