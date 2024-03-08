# Infrastructure for benchmarking

## Pre-requisites

- Some server with public `IP` address and [Docker Compose](https://docs.docker.com/compose/) installed.
- Configured `DNS` records of services listed in [docker-compose.yaml](docker-compose.yaml) ([.env.example](.env.example)).
  - All records should point to the same `IP` address of the server as required by the `traefik` reverse proxy.

## Usage

- Clone this repository.
- Go to cloned repository.
- Copy `.env.example` to `.env` and update the values.
- Run `docker compose up -d --remove-orphans` to start the services.
- Run `docker compose down` to stop the services.
