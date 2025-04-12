# Aggiemenus CI/CD Pipeline README

## Overview

This document describes the Continuous Integration and Continuous Deployment (CI/CD) pipeline for Aggiemenus . The project consists of:

1.  A Next.js frontend application.
2.  A Flask backend application.
3.  A Python scraper script.

The pipeline automates building Docker images, deploying the applications using Docker Compose, exposing them securely via a Cloudflare Tunnel, and running the scraper script on a schedule. The entire process is managed using GitHub Actions running on a self-hosted runner.

## Prerequisites

Before using this pipeline, ensure the following are set up:

1.  **Self-Hosted GitHub Actions Runner:** A runner machine with the label `aggieworks-cicd-sj-1` configured for your repository.
2.  **Docker & Docker Compose:** Docker and Docker Compose must be installed and running on the self-hosted runner machine.
3.  **Cloudflare Account & Tunnel:**
    * A Cloudflare account.
    * A Cloudflare Tunnel created via the Zero Trust dashboard (`Networks` -> `Tunnels`).
    * Your **Tunnel Token** obtained from the Cloudflare dashboard during tunnel setup.
4.  **Docker Hub Account (or other registry):** An account on Docker Hub (or your preferred container registry) if your base images are private or if you intend to push images (though the current workflow primarily builds locally via `docker-compose`).
5.  **Python 3:** Python 3 (specifically version 3.11 as per the workflow) installed on the self-hosted runner for the scraper job.

## Project Structure

The repository should follow a structure similar to this:
```
.
├── .github/
│   └── workflows/
│       └── deploy.yml      # Contains the GitHub Actions workflow
├── frontend/
│   ├── Dockerfile          # Frontend Docker build instructions
│   └── ...               # Next.js app code
├── backend/
│   ├── Dockerfile          # Backend Docker build instructions
│   ├── requirements.txt    # Backend Python dependencies
│   └── ...               # Flask app code
├── docker-compose.yml      # Defines application services and cloudflared
├── requirements.txt        # Scraper Python dependencies (or place in a scripts/ dir)
├── scraper.py              # The scraper script
└── README.md               # This file
```
## Configuration

### 1. GitHub Secrets

Navigate to your repository's `Settings` -> `Secrets and variables` -> `Actions` and add the following secrets:

* `DOCKERHUB_USERNAME`: Your Docker Hub username (or registry username). Needed for logging in, potentially for pulling base images.
* `DOCKERHUB_TOKEN`: Your Docker Hub access token (or registry password/token).
* `TUNNEL_TOKEN`: The **Tunnel Token** obtained from your Cloudflare Tunnel configuration page. This is crucial for the `cloudflared` service in Docker Compose.

### 2. Cloudflare Tunnel Configuration (Dashboard)

Configure your tunnel's ingress rules via the Cloudflare Zero Trust dashboard (`Networks` -> `Tunnels` -> Your Tunnel -> `Public Hostnames`):

* Map your desired public hostname(s) for the frontend to the internal Docker service: `http://frontend:3000`
* Map your desired public hostname(s) for the backend API to the internal Docker service: `http://backend:5000`
    *(Replace `frontend` and `backend` with the exact service names from `docker-compose.yml` if different, and adjust ports if necessary)*

### 3. Dockerfiles

Ensure you have valid `Dockerfile`s in the `./frontend` and `./backend` directories capable of building production-ready images for your Next.js and Flask applications, respectively.

### 4. `docker-compose.yml`

This file defines the services (`frontend`, `backend`, `cloudflared`).
* It uses the images built during the workflow.
* It relies on the `TUNNEL_TOKEN` environment variable to authenticate the `cloudflared` service.
* **Crucially, it defines a bind mount volume for the `backend` service** to map a directory from the host runner (where the scraper writes its file) into the container (where the Flask app reads it). `- /opt/aggimenus/shared-data:/app/shared_data:ro`. **You must update the host path in this definition to match your chosen shared directory.** The `:ro` flag makes the mount read-only inside the container, which is safer if the Flask app only needs to read the data.

### 5. `requirements.txt`

* Ensure `./backend/requirements.txt` lists all Python dependencies for the Flask app.
* Ensure `./requirements.txt` lists all Python dependencies for `scraper.py`.

### 6. Scraper Script (`scraper.py`)

* **Modify your `scraper.py` script** to write its output file(s) to the designated shared directory on the host runner (`/opt/aggimenus/shared-data/menu_data.json`). This path must match the host path used in the `docker-compose.yml` volume mount.

### 7. Flask Application (`backend`)

* Ensure your Flask application code reads the shared data file from the path specified as the *container path* in the `docker-compose.yml` volume mount (e.g., `/app/shared_data/menu_data.json`).


## CI/CD Pipeline Steps (GitHub Actions)

The workflow is defined in `.github/workflows/deploy.yml`.

### Triggers

The workflow runs on:

1.  **Push:** Any push to the `main` branch.
2.  **Schedule:** Daily at midnight UTC (`0 0 * * *`).

### Jobs

1.  **`deploy-frontend` (Runs on Push & Schedule)**
    * Checks out the code.
    * Logs into Docker Hub using secrets.
    * Builds the frontend Docker image using `./frontend/Dockerfile` and tags it `aggiemenus-frontend:latest`.

2.  **`deploy-backend` (Runs on Push & Schedule)**
    * Checks out the code.
    * Logs into Docker Hub using secrets.
    * Builds the backend Docker image using `./backend/Dockerfile` and tags it `aggiemenus-backend:latest`.

3.  **`deploy-services` (Runs on Push & Schedule, after builds)**
    * Checks out the code.
    * Logs into Docker Hub.
    * **Injects `TUNNEL_TOKEN`:
        ```yaml
        - name: Deploy with Docker Compose
          env:
            TUNNEL_TOKEN: ${{ secrets.TUNNEL_TOKEN }}
          run: |
            # ... rest of the script
            docker-compose down
            docker-compose up --build -d
            # ...
        ```
    * Runs `docker-compose down` to stop and remove existing service containers defined in `docker-compose.yml`.
    * Runs `docker-compose up --build -d`. This command:
        * Rebuilds images *if* their build context has changed.
        * Starts all services (`frontend`, `backend`, `cloudflared`) in detached mode (`-d`).
        * **Mounts the shared host directory into the `backend` container** based on the `volumes` definition in `docker-compose.yml`.
        * The `cloudflared` service uses the injected `TUNNEL_TOKEN` to connect to Cloudflare.
    * Optionally prunes unused Docker images and volumes to save space.

4.  **`run-scraper` (Runs ONLY on Schedule)**
    * Checks out the code.
    * Sets up Python 3.11.
    * Installs dependencies from `requirements.txt`.
    * Executes the `scraper.py` script, **which should write its output to the designated shared directory on the host runner.**

## How It Works Summary

1.  A push to `main` triggers the build jobs and the deployment job. The deployment job uses Docker Compose to run the `frontend`, `backend`, and `cloudflared` services. The `backend` service gets the shared data directory mounted from the host.
2.  Cloudflare routes incoming traffic through the tunnel to the `frontend` or `backend` service. The `backend` service can read the file updated by the scraper via the mounted volume.
3.  The daily schedule triggers the `run-scraper` job, which executes the Python script directly on the runner, updating the file in the shared host directory.

