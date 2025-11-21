# SMS-Checker - Operation Registry
This repository is the main entry into this project. It introduces your high-level architecture and links to the other corresponding repositories, so visitors can easily understand your project and find all relevant information. This repository contains a single Docker Compose setup that orchestrates and runs the full application, including:
- `model-service:` Backend Model Service
- `app:` Frontend Web Application

## Repository Structure
The following are the repositories present in this organization:
- [app](https://github.com/doda2025-team21/app) : Frontend Java Spring Boot service providing the UI
- [model-service](https://github.com/doda2025-team21/model-service) : Backend Python microservice that loads and serves the spam classifier
- [lib-version](https://github.com/doda2025-team21/lib-version) : Library dependencies ?? //TODO

## Requirements
The following needs to be installed before running this project:
- Docker
- Docker Compose
Other dependencies such as Python or Java will be installed in respective docker containers.

## Environment Variables Setup
The docker compose uses `.env` file for setting up the environment variables:
```
APP_PORT = 8080
MODEL_PORT = 8081
APP_IMAGE = ghcr.io/doda2025-team21/frontend
MODEL_IMAGE = ghcr.io/doda2025-team21/backend
```
You can change the ports or image versions here.

## How to start the application?
1. Make sure to clone all the repositories using 
```
git clone ....
```
2. Navigate to the appropriate directory 
```
cd operation
```
3. Run the following command: 
```
docker compose up --pull always
```
The following command does the following:
- it will run the latest version of the image from `MODEL_IMAGE`
- Start the backend on [http://localhost:8081](http://localhost:8081) (or replace the port with one mentioned in MODEL_PORT). 
- Start the frontend on [http://localhost:8080/sms](http://localhost:8080/sms) (or replace port with one mentioned in APP_PORT).
An alternative to running the latest version of the image is specifying the version of the you'd like to run using the `TAG_VERSION=***` variable:
```
TAG_VERSION=0.0.1 docker compose up --pull always
```
4. To stop running everything, run the following command:
```
docker compose down
```

## Other handy docker commands

| Action                               | Command                      | Description                            |
| ------------------------------------ | ---------------------------- | -------------------------------------- |
| **Start everything**                 | `docker compose up`          | Starts all services (shows logs)       |
| **Start in background**              | `docker compose up -d`       | Runs services in detached mode         |
| **Stop all running services**        | `docker compose down`        | Stops and removes containers, networks |
| **Rebuild images**                   | `docker compose up --build`  | Rebuilds images before starting        |
| **View logs**                        | `docker compose logs`        | Shows combined logs from all services  |
| **View logs for a specific service** | `docker compose logs app`    | Shows logs only for the app            |
| **Restart one service**              | `docker compose restart app` | Restarts only the app service          |