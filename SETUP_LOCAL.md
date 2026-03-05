# Local Development Setup Guide

This guide walks you through setting up NosCore for local development on your machine using .NET 10.0 and PostgreSQL 18.

---

## Prerequisites

| Tool | Minimum Version | Download |
|------|-----------------|----------|
| .NET SDK | 10.0 | https://dotnet.microsoft.com/download/dotnet/10.0 |
| PostgreSQL | 18 | https://www.postgresql.org/download/ |
| Docker & Docker Compose | 20.10 / 2.0 *(optional)* | https://docs.docker.com/get-docker/ |
| Git | any recent | https://git-scm.com/ |

---

## Option A – Local Setup (without Docker)

### 1. Install .NET 10.0 SDK

Download and install the .NET 10.0 SDK from:  
https://dotnet.microsoft.com/download/dotnet/10.0

Verify the installation:

```bash
dotnet --version
# Expected output: 10.x.x
```

### 2. Install PostgreSQL 18

Download and install PostgreSQL 18 for your platform from:  
https://www.postgresql.org/download/

After installation, make sure the PostgreSQL service is running and the `psql` client is on your `PATH`.

Verify the installation:

```bash
psql --version
# Expected output: psql (PostgreSQL) 18.x
```

### 3. Create the database

Connect to PostgreSQL as the `postgres` superuser and create the `noscore` database:

```bash
psql -U postgres
```

```sql
CREATE DATABASE noscore;
\q
```

### 4. Configure the database connection

The configuration files in the `configuration/` directory use environment-variable substitution with a fallback value (`${VAR,fallback}`).  
The defaults already match the values above, so for a standard local setup no changes are needed.

`configuration/database.yml`:

```yaml
Host: ${DB_HOST,localhost}   # PostgreSQL host (defaults to localhost)
Port: 5432                   # PostgreSQL port
Database: noscore            # Database name
Username: postgres           # Database user
Password: password           # Database password
```

If your local PostgreSQL uses a different user or password, either:
- Edit `configuration/database.yml` directly, **or**
- Export the relevant environment variable before starting the services:

```bash
export DB_HOST=localhost
```

### 5. Apply database migrations

Open the NuGet Package Manager Console in Visual Studio, select the **NosCore.Database** project and run:

```
update-database
```

Or use the .NET CLI from the repository root:

```bash
dotnet ef database update --project src/NosCore.Database
```

### 6. Parse game data

Run the parser to import static game data:

```bash
# From the repository root
dotnet run --project src/NosCore.Parser
```

### 7. Start the services

Use the helper scripts in the `scripts/` directory, or start each service individually:

```bash
# Master server
dotnet run --project src/NosCore.MasterServer

# World server (in a separate terminal)
dotnet run --project src/NosCore.WorldServer

# Login server (in a separate terminal)
dotnet run --project src/NosCore.LoginServer
```

---

## Option B – Docker-based Setup

This option runs PostgreSQL 18 and all NosCore services inside Docker containers.

### 1. Prerequisites

Make sure Docker and Docker Compose are installed and running.

### 2. Build the application binaries

```bash
dotnet publish -c Release -r linux-musl-x64 --self-contained false \
  -o build/net10.0/linux-musl-x64 NosCore.sln
```

### 3. Start all services

```bash
docker compose up -d
```

Docker Compose will:
- Pull the `postgres:18-alpine` image
- Start the database with a health check
- Wait until the database is healthy before starting `master`, `world`, and `login`

### 4. Apply database migrations

Once the `db` container is healthy, apply migrations from the host machine:

```bash
dotnet ef database update --project src/NosCore.Database \
  --connection "Host=localhost;Port=5432;Database=noscore;Username=postgres;Password=password"
```

### 5. View logs

```bash
docker compose logs -f
```

### 6. Stop services

```bash
docker compose down
```

To also remove the database volume:

```bash
docker compose down -v
```

---

## Environment Variable Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_HOST` | `localhost` | PostgreSQL hostname |
| `WEBAPI_HOST` | `http://localhost` | Master server Web API base URL |
| `MASTER_HOST` | `http://localhost` | Master server base URL (used by world/login) |
| `WEBAPI_PORT` | `5001` | World server Web API port |
| `WORLD_PORT` | `1337` | World server game port |
| `LOGIN_PORT` | `4000` | Login server game port |
| `HOST` | `127.0.0.1` | Bind address for game servers |
| `PORT` | `5000` | Master server port |

---

## Connection String Format

NosCore uses individual fields in `database.yml` rather than a single connection string.  
The equivalent Npgsql connection string is:

```
Host=<DB_HOST>;Port=5432;Database=noscore;Username=postgres;Password=password
```

---

## Troubleshooting

### `pg_isready` fails / cannot connect to PostgreSQL

- Ensure the PostgreSQL service is running:
  ```bash
  # Linux (systemd)
  sudo systemctl status postgresql

  # macOS (Homebrew)
  brew services list
  ```
- Check that the port is not blocked by a firewall or another process:
  ```bash
  netstat -an | grep 5432
  ```

### `update-database` fails with authentication error

- Verify the credentials in `configuration/database.yml` match your PostgreSQL user.
- For PostgreSQL 18 the default authentication method is `scram-sha-256`.  
  Ensure your client library (Npgsql) is up to date.

### Docker containers exit immediately

- Check logs: `docker compose logs db`
- If the `postgres` volume already exists with data from an older version, remove it:
  ```bash
  docker compose down -v
  docker compose up -d
  ```

### Port conflicts

If a local PostgreSQL instance is already running on port 5432, either stop it before starting Docker Compose, or change the host port mapping in `docker-compose.yml`:

```yaml
ports:
  - 5433:5432   # map container port 5432 to host port 5433
```

Then update `DB_HOST` and the port in `configuration/database.yml` accordingly.

### `dotnet ef` command not found

Install the EF Core tools globally:

```bash
dotnet tool install --global dotnet-ef
```

---

## Verification

After completing setup, verify that all services are reachable:

| Service | URL / Port | Expected response |
|---------|-----------|-------------------|
| Master Web API | http://localhost:5000 | HTTP 200 / JSON |
| World Web API | http://localhost:5001 | HTTP 200 / JSON |
| Login server | TCP 4000 | accepts connections |
| World server | TCP 1337 | accepts connections |

You can use `curl` to check the Web API endpoints:

```bash
curl -I http://localhost:5000
curl -I http://localhost:5001
```
