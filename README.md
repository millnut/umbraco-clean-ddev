# Umbraco CMS - DDEV Local Development

This project runs Umbraco CMS on .NET 10.0, using DDEV to manage the local Docker environment. It uses Paul Seal's [Clean](https://github.com/prjseal/Clean) Starter Kit.

## Prerequisites

- [DDEV](https://ddev.readthedocs.io/en/stable/) installed
- Docker (Docker Desktop or OrbStack)

## Getting Started

```bash
ddev start
```

A `pre-start` hook in `config.yaml` copies `appsettings.Local.json.example` to `appsettings.Local.json` if it is missing, so a fresh clone needs no setup step. On first boot Umbraco will run its unattended install (creating the database schema and default admin user).

Default admin credentials (configured in `appsettings.Local.json`):
- Email: `admin@example.com`
- Password: `1234567890`

On that first boot Umbraco logs a `Cannot open database "UmbracoDb"` warning with error number 4060. It appears because the database does not exist until the unattended install creates it, and it does not recur.

## How It Works

The project uses DDEV's `generic` type, so DDEV starts no nginx or PHP-FPM of its own. The stock `ddev-webserver` image is kept and extended: `.ddev/web-build/Dockerfile.dotnet` adds the .NET SDK and SQL Server tooling on top of it rather than replacing it.

| Service | Notes |
|---------|-------|
| `web` | The stock DDEV web container plus .NET. `web_extra_daemons` in `config.yaml` runs `dotnet watch` under the container's supervisord, so Kestrel gets restart-on-failure for free. Kestrel binds port 80 directly, which is where the DDEV router sends this project's traffic. |
| `sqlserver` | Azure SQL Edge, used because SQL Server proper publishes no ARM64 image. It speaks the same TDS protocol, so Umbraco cannot tell the difference. |
| `adminer` | Web-based database UI. |

DDEV's router handles TLS termination and forwards plain HTTP to Kestrel, which is why `ASPNETCORE_FORWARDEDHEADERS_ENABLED` is set.

Only `sqlserver` and `adminer` are defined in `docker-compose.umbraco.yaml`. The web container is configured entirely through `config.yaml`, so DDEV keeps ownership of its lifecycle.

## Configuration

Database credentials live in `.ddev/.env`, which DDEV feeds to both docker-compose interpolation and every container's environment. One definition there covers the `sqlserver` service, Adminer, the `bacpac-*` commands, and Umbraco's connection string. The file is committed on purpose: it holds a fixed local dev credential rather than a secret, so a fresh clone works without extra setup.

`appsettings.Local.json` is git-ignored and exists for per-developer overrides such as logging levels. `Program.cs` registers it last:

```csharp
builder.Configuration.AddJsonFile("appsettings.Local.json", optional: true, reloadOnChange: true);
```

Because that line runs after `CreateBuilder` has already added the environment variable provider, this file wins over the environment. Do not add a `ConnectionStrings` block to it: doing so would quietly override the value from `.ddev/.env`. The example template is at `MyProject/appsettings.Local.json.example`.

## Commands

| Command | Description |
|---------|-------------|
| `ddev dotnet <args>` | Run dotnet CLI in the project directory |
| `ddev bacpac-export [filename]` | Export database to a `.bacpac` file |
| `ddev bacpac-import <filename>` | Drop and reimport database from a `.bacpac` file |
| `ddev adminer` | Open Adminer in browser |

## Database Backup / Restore

Export the current database:
```bash
ddev bacpac-export
```
This writes `UmbracoDb.bacpac` to the project root.

Import from a bacpac (drops the existing database first):
```bash
ddev bacpac-import UmbracoDb.bacpac
```

Place the `.bacpac` file in the project root before importing.

## Architecture

```
.ddev/
├── .env                           # SA password and connection string
├── config.yaml                    # Project config, web container, Kestrel daemon
├── docker-compose.umbraco.yaml    # sqlserver and adminer services
├── docker-compose.adminer.yaml    # Adminer add-on (modified, see below)
├── web-build/
│   └── Dockerfile.dotnet          # Adds .NET SDK and mssql tools to ddev-webserver
└── commands/
    ├── host/
    │   └── adminer                # Adminer launcher (from the add-on)
    └── web/
        ├── bacpac-export          # Database export
        ├── bacpac-import          # Database import (drops existing first)
        └── dotnet                 # dotnet CLI passthrough
```

### Adminer add-on

Both files the `ddev/ddev-adminer` add-on provides are committed here, so `ddev add-on get` never needs to run and a clone starts with Adminer already working.

`docker-compose.adminer.yaml` carries one modification: the default `depends_on: [db]` is removed because this project omits the DDEV `db` container (SQL Server runs as its own service). The connection defaults are set in `docker-compose.umbraco.yaml` instead, pointing Adminer at the `sqlserver` service.

Neither committed copy carries a `#ddev-generated` marker, so DDEV treats them as yours and will neither overwrite them on a future `ddev add-on get` nor delete them on `ddev add-on remove`.
