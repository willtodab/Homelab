# Docker Application Host

A dedicated Ubuntu Server host running Docker Engine and Docker Compose for self-hosted applications and homelab services.

## Purpose

Docker provides a consistent way to deploy, update, isolate, and troubleshoot services without installing every application directly onto the host operating system.

The Docker host is kept separate from core network infrastructure such as DNS and DHCP. This allows application containers to be restarted, updated, or rebuilt without disrupting basic network connectivity.

## Current Responsibilities

The Docker host currently supports services including:

- Traefik reverse proxy
- Plex Media Server
- Home Assistant
- Additional experimental and self-hosted applications
- Persistent application configuration and media storage
- Tailscale-based remote administration

Core network services such as Pi-hole and DHCP run on separate dedicated hardware.

## Architecture

```text
Clients
   |
Local Network
   |
Docker Host
   |
   +-- Traefik
   |     +-- Routes web requests to internal services
   |
   +-- Home Assistant
   |     +-- Home automation and integrations
   |
   +-- Plex
   |     +-- Local media streaming
   |
   +-- Additional Containers
         +-- Monitoring
         +-- Development services
         +-- Future applications
```

## Docker Engine

Docker Engine runs and manages the containers.

Useful checks:

```bash
docker --version
docker info
systemctl status docker
systemctl is-enabled docker
```

Start Docker:

```bash
sudo systemctl start docker
```

Restart Docker:

```bash
sudo systemctl restart docker
```

View recent daemon logs:

```bash
sudo journalctl -u docker -n 100 --no-pager
```

Follow daemon logs:

```bash
sudo journalctl -fu docker
```

## Docker Compose

Docker Compose defines multi-container applications using YAML files.

Check the installed version:

```bash
docker compose version
```

Validate the current Compose configuration:

```bash
docker compose config
```

Start or update a Compose project:

```bash
docker compose up -d
```

Show project status:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs
```

Follow logs:

```bash
docker compose logs -f
```

Pull newer images:

```bash
docker compose pull
```

Recreate services using the current configuration and images:

```bash
docker compose up -d
```

Stop and remove the project containers:

```bash
docker compose down
```

Persistent volumes and bind-mounted files should be reviewed before removing or rebuilding services.

## Container Management

List running containers:

```bash
docker ps
```

List all containers:

```bash
docker ps -a
```

Display a compact service overview:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
```

Inspect a container:

```bash
docker inspect CONTAINER_NAME
```

View container logs:

```bash
docker logs CONTAINER_NAME
```

Follow container logs:

```bash
docker logs -f CONTAINER_NAME
```

Open a shell inside a container:

```bash
docker exec -it CONTAINER_NAME /bin/sh
```

Use Bash when the image includes it:

```bash
docker exec -it CONTAINER_NAME /bin/bash
```

View live resource usage:

```bash
docker stats
```

## Networking

Docker uses virtual networks to connect containers while isolating workloads from one another.

List Docker networks:

```bash
docker network ls
```

Inspect a network:

```bash
docker network inspect NETWORK_NAME
```

Show each running container and its attached networks:

```bash
docker inspect \
  --format '{{.Name}}: {{range $name, $config := .NetworkSettings.Networks}}{{$name}} {{end}}' \
  $(docker ps -q)
```

List services listening on the host:

```bash
sudo ss -lntup
```

### Reverse Proxy

Traefik provides centralized HTTP and HTTPS entrypoints for selected applications.

Instead of exposing every web service independently, Traefik can route requests based on hostnames and configuration labels.

This provides:

- Centralized routing
- Consistent service access
- Reduced direct port exposure
- A foundation for future TLS configuration
- Easier organization as more applications are added

## Persistent Storage

Containers are disposable, but application data is not.

Persistent data is stored using:

- Bind mounts
- Docker named volumes
- Host storage directories
- Application-specific configuration directories

List Docker volumes:

```bash
docker volume ls
```

Inspect a volume:

```bash
docker volume inspect VOLUME_NAME
```

Show Docker disk usage:

```bash
docker system df
```

Show detailed disk usage:

```bash
docker system df -v
```

Check host filesystem usage:

```bash
df -h
```

Check inode usage:

```bash
df -i
```

Destructive cleanup commands should not be run without reviewing their effects:

```bash
docker system prune
docker volume prune
```

An unused volume may still contain important application data.

## Service Troubleshooting

### Container is not running

```bash
docker ps -a
docker logs CONTAINER_NAME
docker inspect CONTAINER_NAME
```

Questions to ask:

- Did the container exit?
- What was its exit code?
- Is a required file or volume missing?
- Is another process already using its port?
- Can the container reach DNS and the network?
- Does the application have permission to access its files?

### Container repeatedly restarts

```bash
docker ps -a
docker logs --tail 200 CONTAINER_NAME
docker inspect \
  --format '{{.State.Status}} {{.State.ExitCode}} {{.State.Error}}' \
  CONTAINER_NAME
```

### Port conflict

```bash
sudo ss -lntup
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

Determine whether the port belongs to:

- A host systemd service
- A published Docker port
- A host-networked container
- Traefik
- Another reverse proxy or application

### Service works directly but not through Traefik

Check:

- Container health
- Direct application access
- Docker network membership
- Traefik labels
- Routing rules
- Traefik logs
- Trusted-proxy settings
- Local DNS records

### DNS failure inside a container

Inspect the container resolver configuration:

```bash
docker exec CONTAINER_NAME cat /etc/resolv.conf
```

Test name resolution:

```bash
docker exec CONTAINER_NAME getent hosts example.com
```

Compare it with the host:

```bash
dig example.com
```

## Security

The Docker environment follows several basic security practices:

- Remote administration uses SSH keys
- Tailscale provides private remote connectivity
- Management services are not intentionally exposed to the public internet
- Traefik centralizes web-service routing
- Secrets are excluded from public Git repositories
- Private infrastructure details are stored in ignored local documentation
- Core DNS and DHCP services run outside the Docker host
- Persistent data is separated from disposable containers

Never publish:

- Passwords
- API keys
- Authentication tokens
- Private SSH keys
- Raw environment files
- Database credentials
- Cloud provider tokens
- VPN authentication keys
- Private certificates

Public Compose examples should reference environment variables without including their actual values.

## Backup Priorities

Important items to back up include:

- Compose files
- Application configuration directories
- Environment files
- Persistent Docker volumes
- Reverse-proxy configuration
- Home automation configuration
- Media-server metadata
- Documentation
- Recovery procedures

A backup is not fully trusted until a restore has been tested.

## Recovery Workflow

When a service fails:

1. Confirm the host is reachable.
2. Check host addressing and routing.
3. Confirm DNS is working.
4. Check the Docker daemon.
5. List running and stopped containers.
6. Inspect listening ports.
7. Read the affected container logs.
8. Validate the Compose configuration.
9. Inspect volumes and permissions.
10. Recreate only the affected service when possible.

Useful commands:

```bash
ip -4 -br addr
ip route
systemctl status docker
sudo ss -lntup
docker ps -a
sudo journalctl -u docker -n 100 --no-pager
```

## Troubleshooting Philosophy

Docker troubleshooting is approached by identifying the subsystem responsible for the failure.

Examples:

- Container lifecycle problem → Docker or application logs
- Port conflict → Host networking
- Name-resolution failure → DNS
- Reverse-proxy failure → Traefik routing or trusted proxies
- Missing data → Volumes, bind mounts, or permissions
- Service fails after reboot → Restart policy or dependency ordering
- Host unreachable → Physical network, routing, or operating system

The goal is to gather evidence before changing configuration.

## Future Work

- Inventory every active Compose project
- Document each service separately
- Create tested backups for persistent data
- Add Prometheus and Grafana monitoring
- Add service-health alerts
- Review container restart policies
- Review CPU and memory limits
- Establish an image-update strategy
- Create sanitized Compose examples
- Add automated configuration validation
- Test restoring a noncritical service
- Evaluate which workloads should migrate to Kubernetes
- Use Kubernetes for monitoring, development, and selected applications
