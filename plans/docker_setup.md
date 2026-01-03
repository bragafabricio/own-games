# Docker Setup Guide for Game Information System

This guide explains how to set up and use Docker for the PostgreSQL database in your Game Information System.

## Prerequisites

1. Install Docker and Docker Compose on your system:
   - [Docker Desktop](https://www.docker.com/products/docker-desktop/) for Windows/Mac
   - For Linux: [Docker Engine](https://docs.docker.com/engine/install/) and [Docker Compose](https://docs.docker.com/compose/install/)

## Docker Compose Configuration

Create a `docker-compose.yml` file in your project root with the following content:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16
    container_name: games-postgres
    environment:
      POSTGRES_DB: gamesdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

## Starting the Database

1. Open a terminal in your project root directory
2. Run the following command to start the PostgreSQL container:

```bash
docker-compose up -d
```

The `-d` flag runs the container in detached mode (background).

## Verifying the Database

To verify that your PostgreSQL container is running:

```bash
docker ps
```

You should see output similar to:

```
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS                    NAMES
abc123def456   postgres:16   "docker-entrypoint.s…"   2 minutes ago   Up 2 minutes   0.0.0.0:5432->5432/tcp   games-postgres
```

## Connecting to the Database

You can connect to the PostgreSQL database using any PostgreSQL client with these credentials:

- Host: localhost
- Port: 5432
- Database: gamesdb
- Username: postgres
- Password: postgres

### Using psql (PostgreSQL CLI)

If you have the PostgreSQL client tools installed, you can connect with:

```bash
psql -h localhost -p 5432 -U postgres -d gamesdb
```

Or you can use the Docker container's psql:

```bash
docker exec -it games-postgres psql -U postgres -d gamesdb
```

## Stopping the Database

To stop the PostgreSQL container:

```bash
docker-compose down
```

To stop and remove all data (volumes):

```bash
docker-compose down -v
```

## Database Management

### Viewing Logs

To view the PostgreSQL logs:

```bash
docker logs games-postgres
```

### Backing Up the Database

To create a backup of your database:

```bash
docker exec -t games-postgres pg_dump -U postgres gamesdb > backup.sql
```

### Restoring from Backup

To restore from a backup:

```bash
cat backup.sql | docker exec -i games-postgres psql -U postgres -d gamesdb
```

## Troubleshooting

### Port Conflict

If you see an error like "port 5432 already in use", you may have PostgreSQL already running on your system. You can either:

1. Stop your local PostgreSQL service, or
2. Change the port mapping in docker-compose.yml to use a different port:

```yaml
ports:
  - "5433:5432"
```

Then update your application.properties to use port 5433.

### Container Not Starting

If the container fails to start, check the logs:

```bash
docker logs games-postgres
```

Common issues include:
- Insufficient disk space
- Permission problems with mounted volumes
- Conflicting PostgreSQL data in the volume

### Connection Issues

If your Spring Boot application can't connect to the database, verify:
1. The container is running (`docker ps`)
2. Your application.properties has the correct connection details
3. No firewall is blocking the connection