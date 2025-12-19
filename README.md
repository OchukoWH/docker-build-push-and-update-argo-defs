# Docker Build, Push, and ArgoCD Update Pipeline

This repository contains a CI/CD pipeline that automatically builds Docker images, pushes them to a private Docker Hub repository, and updates the image references in Kubernetes manifests stored in an ArgoCD GitOps repository.

## Overview

When code is pushed to the `main` branch, this GitHub Actions workflow:

1. **Builds** three Docker images:
   - `vprofiledb2` - Database container
   - `vprofileapp2` - Application container  
   - `vprofileweb2` - Web/NGINX container

2. **Pushes** the images to Docker Hub with two tags:
   - `latest` - Points to the most recent build
   - `{short-sha}` - Unique tag based on Git commit SHA (e.g., `a1b2c3d`)

3. **Updates** the Kubernetes manifests in the ArgoCD repository:
   - Updates `vprofile/appdeploy.yaml` with the new application image tag
   - Updates `vprofile/dbdeploy.yaml` with the new database image tag
   - Updates `vprofile/webdeploy.yaml` with the new web image tag
   - Commits and pushes the changes to trigger ArgoCD sync

## Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   GitHub Repo   │───►│  GitHub Actions │───►│   Docker Hub    │
│   (This Repo)   │    │  (Self-Hosted)  │    │  (Private Repo) │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │  ArgoCD Repo     │
                       │  (GitOps)        │
                       └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   Kubernetes    │
                       │   Cluster       │
                       └─────────────────┘
```

## Prerequisites

Before setting up this pipeline, ensure you have:

1. **Docker Hub Account** with a private repository
2. **GitHub Repository** with Actions enabled
3. **Self-Hosted Ubuntu EC2 Runner** configured
4. **ArgoCD Repository Access** - This repository needs permission to push to the ArgoCD GitOps repository
5. **GitHub Personal Access Token** with `repo` scope for updating the ArgoCD repository

## Setup Instructions

### Step 1: Set Up Self-Hosted Runner on Ubuntu EC2

1. **Launch an Ubuntu EC2 instance** (recommended: t2.medium or larger)

2. **Install Docker** on the EC2 instance:
   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io
   sudo systemctl start docker
   sudo systemctl enable docker
   sudo usermod -aG docker ubuntu
   ```

3. **Install GitHub Actions Runner**:
   ```bash
   # Create a folder
   mkdir actions-runner && cd actions-runner
   
   # Download the latest runner package
   curl -o actions-runner-linux-x64-2.311.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
   
   # Extract the installer
   tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz
   
   # Configure the runner (you'll get a token from GitHub)
   ./config.sh --url https://github.com/YOUR_USERNAME/YOUR_REPO --token YOUR_RUNNER_TOKEN
   
   # Install as a service
   sudo ./svc.sh install
   sudo ./svc.sh start
   ```

4. **Verify the runner** appears in your repository:
   - Go to `Settings` → `Actions` → `Runners`
   - You should see your self-hosted runner listed

### Step 2: Configure GitHub Secrets

Navigate to your repository: `Settings` → `Secrets and variables` → `Actions` → `New repository secret`

Add the following secrets:

#### Required Secrets

| Secret Name | Description | How to Get |
|------------|-------------|------------|
| `DOCKERHUB_USERNAME` | Your Docker Hub username | Your Docker Hub account username |
| `DOCKERHUB_TOKEN` | Docker Hub access token | Docker Hub → Account Settings → Security → New Access Token |
| `GIT_TOKEN` | GitHub Personal Access Token | GitHub → Settings → Developer settings → Personal access tokens → Generate new token (classic) with `repo` scope |

#### Creating Docker Hub Access Token

1. Log in to [Docker Hub](https://hub.docker.com/)
2. Go to **Account Settings** → **Security** → **New Access Token**
3. Create a token with **Read & Write** permissions
4. Copy and save the token (you won't be able to see it again!)

#### Creating GitHub Personal Access Token

1. Go to GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**
2. Click **Generate new token (classic)**
3. Give it a descriptive name (e.g., "ArgoCD Update Token")
4. Select the `repo` scope (full control of private repositories)
5. Click **Generate token**
6. Copy the token immediately (you won't be able to see it again!)

### Step 3: Grant Repository Access to ArgoCD Repo

The workflow needs permission to push to the ArgoCD repository (`OchukoWH/automatic-updates-argo-defs`).

**Option 1: Using Personal Access Token (Recommended)**
- Use the `GIT_TOKEN` secret created above
- Ensure the token has access to the ArgoCD repository

**Option 2: Using GitHub App or Deploy Key**
- Create a GitHub App or deploy key with write access to the ArgoCD repository
- Configure it as a secret in this repository

### Step 4: Verify Docker Hub Repositories

Ensure these repositories exist in your Docker Hub account (or update the workflow file with your repository names):

- `{DOCKERHUB_USERNAME}/vprofiledb2`
- `{DOCKERHUB_USERNAME}/vprofileapp2`
- `{DOCKERHUB_USERNAME}/vprofileweb2`

## How It Works

### Workflow Trigger

The workflow triggers automatically on **every push to the `main` branch**.

### Workflow Steps

1. **Checkout Code**: Checks out the repository code
2. **Setup Docker Buildx**: Sets up Docker Buildx for optimized builds
3. **Login to Docker Hub**: Authenticates using `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`
4. **Extract Short SHA**: Extracts the first 7 characters of the Git commit SHA
5. **Build and Push Images**: Builds and pushes three Docker images:
   - Database image: `vprofiledb2:latest` and `vprofiledb2:{sha}`
   - Application image: `vprofileapp2:latest` and `vprofileapp2:{sha}`
   - Web image: `vprofileweb2:latest` and `vprofileweb2:{sha}`
6. **Update ArgoCD Manifests**:
   - Clones the ArgoCD repository (`OchukoWH/automatic-updates-argo-defs`)
   - Updates `vprofile/appdeploy.yaml` with the new application image tag
   - Updates `vprofile/dbdeploy.yaml` with the new database image tag
   - Updates `vprofile/webdeploy.yaml` with the new web image tag
   - Commits and pushes the changes

### Image Tagging Strategy

Each build creates **two tags** for each image:

- **`latest`**: Always points to the most recent build (overwrites previous `latest`)
- **`{short-sha}`**: Unique tag based on Git commit SHA (e.g., `a1b2c3d`)

**Why two tags?**
- `latest` provides easy access to the newest build
- `{short-sha}` preserves all previous images for rollback or version tracking

### ArgoCD Manifest Updates

The workflow automatically updates these files in the ArgoCD repository:

- **`vprofile/appdeploy.yaml`**: Updates the `image` field for the application deployment
- **`vprofile/dbdeploy.yaml`**: Updates the `image` field for the database deployment
- **`vprofile/webdeploy.yaml`**: Updates the `image` field for the web/NGINX deployment

The updated image references use the commit SHA tag (e.g., `username/vprofileapp2:a1b2c3d`), ensuring ArgoCD deploys the exact version that was built.

## Docker Images

### Database Image (`vprofiledb2`)
- **Dockerfile**: `Docker-files/db/Dockerfile`
- **Context**: `Docker-files/db/`
- **Base**: MySQL

### Application Image (`vprofileapp2`)
- **Dockerfile**: `Docker-files/app/multistage/Dockerfile`
- **Context**: Root directory
- **Base**: Multi-stage build (Maven → Tomcat)

### Web Image (`vprofileweb2`)
- **Dockerfile**: `Docker-files/web/Dockerfile`
- **Context**: `Docker-files/web/`
- **Base**: NGINX

## Troubleshooting

### Runner Not Picking Up Jobs

1. Check if the runner is online:
   - Go to `Settings` → `Actions` → `Runners`
   - Verify the runner shows as "Idle" or "Active"

2. Check runner logs:
   ```bash
   # On the EC2 instance
   sudo journalctl -u actions.runner.* -f
   ```

3. Restart the runner service:
   ```bash
   cd actions-runner
   sudo ./svc.sh stop
   sudo ./svc.sh start
   ```

### Docker Build Failures

1. **Check Docker is running**:
   ```bash
   sudo systemctl status docker
   ```

2. **Check disk space**:
```bash
   df -h
   ```

3. **Check Docker Hub credentials**:
   - Verify `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` secrets are correct
   - Ensure the token has read/write permissions

### ArgoCD Update Failures

1. **Check Git Token**:
   - Verify `GIT_TOKEN` secret is valid
   - Ensure the token has `repo` scope and access to the ArgoCD repository

2. **Check Repository Access**:
   - Verify the token can push to `OchukoWH/automatic-updates-argo-defs`
   - Check if the repository exists and is accessible

3. **Check Workflow Logs**:
   - View the workflow run logs in GitHub Actions
   - Look for specific error messages in the "Update ArgoCD manifests" step

### Permission Denied Errors

If you see permission errors:

1. **Docker permission**:
   ```bash
   sudo usermod -aG docker ubuntu
   # Log out and log back in
   ```

2. **File permissions**:
   ```bash
   sudo chown -R ubuntu:ubuntu /home/ubuntu/actions-runner
   ```

## Security Considerations

1. **Secrets**: Never commit secrets to the repository. Always use GitHub Secrets.
2. **Docker Hub**: Use access tokens instead of passwords for better security.
3. **Git Token**: Use a token with minimal required permissions (`repo` scope only).
4. **Self-Hosted Runner**: Keep the EC2 instance updated with security patches.
5. **Network**: Consider restricting network access to the runner if possible.

## Monitoring

- **Workflow Runs**: View in `Actions` tab of your GitHub repository
- **Docker Hub**: Check your Docker Hub repositories for pushed images
- **ArgoCD**: Monitor ArgoCD for automatic syncs triggered by manifest updates

## Support

For issues or questions:
1. Check the workflow logs in GitHub Actions
2. Review the troubleshooting section above
3. Check runner logs on the EC2 instance
4. Verify all secrets are correctly configured

## License

[Add your license here]
