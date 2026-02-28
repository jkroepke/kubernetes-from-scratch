# TCP Load Balancer with Traefik

This directory contains a Traefik-based TCP load balancer configuration for the Kubernetes cluster.

## Overview

The load balancer routes traffic to:
- **Port 6443**: Kubernetes API Server (controller1, controller2, controller3)
- **Port 80**: HTTP Ingress (worker1)
- **Port 443**: HTTPS Ingress (worker1)

## Usage

Start the load balancer:

```bash
cd docker/lb
docker-compose up -d
```

Stop the load balancer:

```bash
docker-compose down
```

View logs:

```bash
docker-compose logs -f
```

## Configuration

- **docker-compose.yaml**: Defines the Traefik container with host networking
- **traefik.yml**: TCP routing configuration for load balancing

## Requirements

- Docker and Docker Compose
- Host names (controller1, controller2, controller3, worker1) must be resolvable to their respective IP addresses
- The load balancer runs with `network_mode: host` to directly bind to ports 80, 443, and 6443

## Traefik Dashboard

The Traefik dashboard is available at:
- http://localhost:8080/dashboard/

This provides visibility into the configured routers, services, and health status.
