# Docker Compose Configuration for Dremio, MinIO, Nessie, Postgres, and MongoDB

This Docker Compose file sets up a local environment for running Dremio, MinIO, Nessie, Postgres, and MongoDB services. It creates a shared network for all services, along with defined ports and environment configurations. 

## Services Overview

### 1. **Dremio**
   - **Image**: `dremio/dremio-oss:latest`
   - **Container Name**: `dremio`
   - **Ports**:
     - `9047:9047`: Dremio Web UI
     - `31010:31010`: Dremio JDBC/ODBC communication
     - `32010:32010`: Dremio Arrow Flight communication
     - `45678:45678`: Dremio internal communication
   - **Environment**:
     - `DREMIO_JAVA_SERVER_EXTRA_OPTS`: Extra JVM options for Dremio. Sets the `paths.dist` directory for Dremio to `/opt/dremio/data/dist`.
   - **Depends on**:
     - `minio`: Object storage service for Dremio to access.
     - `nessie`: Versioned data lake storage.
     - `postgres`: Used as the backend database for Nessie.
     - `mongo`: NoSQL database that Dremio can query.
   - **Network**: `mongo-postgres-dremio`.

### 2. **MinIO**
   - **Image**: `minio/minio`
   - **Container Name**: `minio`
   - **Environment**:
     - `MINIO_ROOT_USER`: Sets the MinIO admin username to `admin`.
     - `MINIO_ROOT_PASSWORD`: Sets the MinIO admin password to `password`.
   - **Command**:
     - Runs the MinIO server and creates buckets `datalake` and `datalakehouse` using the MinIO client (`mc`).
   - **Ports**:
     - `9000:9000`: MinIO S3 API endpoint.
     - `9001:9001`: MinIO web console.
   - **Healthcheck**:
     - A health check is configured to ensure that the MinIO service is running properly, checking the MinIO health endpoint.
     - **Interval**: 30 seconds.
     - **Timeout**: 20 seconds.
     - **Retries**: 3.
   - **Volumes**: No volumes defined, data is stored inside the container.
   - **Network**: `mongo-postgres-dremio`.

### 3. **Nessie**
   - **Image**: `projectnessie/nessie`
   - **Container Name**: `nessie`
   - **Environment**:
     - `QUARKUS_PROFILE`: Configures Nessie to use the `postgres` profile.
     - `QUARKUS_DATASOURCE_JDBC_URL`: Configures the connection to the Postgres instance using the URL `jdbc:postgresql://postgres:5432/nessie`.
     - `QUARKUS_DATASOURCE_USERNAME`: Nessie connects to Postgres using the `nessie` user.
     - `QUARKUS_DATASOURCE_PASSWORD`: Password for the `nessie` user in Postgres is set to `nessie`.
   - **Ports**:
     - `19120:19120`: Exposes the Nessie API on port 19120.
   - **Depends on**:
     - `postgres`: Nessie depends on Postgres for storing metadata.
   - **Network**: `mongo-postgres-dremio`.

### 4. **Postgres**
   - **Image**: `postgres:13`
   - **Container Name**: `postgres`
   - **Environment**:
     - `POSTGRES_USER`: Sets the username for the Postgres database to `nessie`.
     - `POSTGRES_PASSWORD`: Sets the password for the `nessie` user to `nessie`.
     - `POSTGRES_DB`: Creates a database named `nessie`.
   - **Ports**:
     - `5435:5432`: Exposes the Postgres database on port 5435 of the host (mapped from 5432 of the container).
   - **Volumes**:
     - Mounts the local directory `./seed/postgres` to `/docker-entrypoint-initdb.d/` for seeding the Postgres database with initial data.
   - **Network**: `mongo-postgres-dremio`.

### 5. **MongoDB**
   - **Image**: `mongo:4.4`
   - **Container Name**: `mongo`
   - **Environment**:
     - `MONGO_INITDB_ROOT_USERNAME`: Sets the MongoDB root username to `root`.
     - `MONGO_INITDB_ROOT_PASSWORD`: Sets the MongoDB root password to `example`.
   - **Ports**:
     - `27017:27017`: Exposes the MongoDB service on the default MongoDB port.
   - **Volumes**:
     - Mounts the local directory `./seed/mongo` to `/docker-entrypoint-initdb.d/` for seeding the MongoDB database with initial data.
   - **Network**: `mongo-postgres-dremio`.

## Networks

- **mongo-postgres-dremio**: A custom bridge network that connects all services. It allows communication between the Dremio, MinIO, Nessie, Postgres, and MongoDB containers.

## Volumes

- **minio-data**: A volume where MinIO stores its object data (defined but not used explicitly in this compose file).
- **postgres-data**: A volume where Postgres stores its database data (not explicitly defined in this compose file).
- **mongo-data**: A volume where MongoDB stores its database data (not explicitly defined in this compose file).

## Running the Services

To start the services, run the following command:

```bash
docker-compose up -d
```