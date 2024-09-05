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

This command will start all the containers in detached mode. Once running, you can access the following:

- Dremio UI: http://localhost:9047
- MinIO Console: http://localhost:9001
- Nessie API: http://localhost:19120
- Postgres: Available on port 5435
- MongoDB: Available on port 27017

## Stopping the Services

To stop the services, run:

```bash
docker-compose down
```

If you want to remove all data volumes (i.e., reset the environment):

```bash
docker-compose down -v
```

# Connecting Nessie, MinIO, Postgres, and MongoDB to Dremio from the Dremio Web UI

Once all services are up and running via Docker Compose, you can connect these sources to Dremio from the Dremio Web UI.

## Prerequisites

Ensure the services are running and accessible:

- Dremio UI: `http://localhost:9047`
- MinIO: `http://localhost:9001`
- Nessie API: `http://localhost:19120`
- Postgres: Accessible via `localhost:5435`
- MongoDB: Accessible via `localhost:27017`

## 1. Connecting to Nessie (Versioned Data Lake)

Nessie is a version control system for your data lake and can be connected to Dremio using the built-in Nessie support.

### Steps:

1. **Log in to Dremio**: Open your browser and go to `http://localhost:9047`. Log in with your Dremio credentials.
   
2. **Navigate to Sources**:
   - On the Dremio Web UI, click on the **Sources** tab in the left-hand sidebar.
   
3. **Add Nessie as a Source**:
   - Click **+ Source** in the upper-right corner.
   - Select **Nessie** from the list of available source types.

4. **Configure Nessie Source**:
   - **Name**: Give your Nessie source a name (e.g., `nessie`).
   - **Nessie REST API URL**: Enter `http://nessie:19120` (the API URL exposed by the Nessie container, based on the `container_name` defined in the `docker-compose.yml` file).
   - **Authentication**: Choose `None`

5. **Configure Nessie Storage**:
    - set the warehouse address to the name of the bucket in MinIO (datalakehouse)
    - access key and secret key are the same as the ones used to access MinIO defined  in the docker-compose.yml file. (admin/password)
    - set the following custom parameters:
        - `fs.s3a.path.style.access` to true
        - `fs.s3a.endpoint` to the endpoint of MinIO (minio:9000)
        - `dremio.s3.compat` to true
    
6. **Click Save**: Once the configuration is set, click **Save**. Dremio will now be connected to Nessie, and you will be able to read and write versioned data using Iceberg tables managed by Nessie.

---

## 2. Connecting to MinIO (Object Storage)

MinIO provides S3-compatible object storage, and you can add it as an external source in Dremio.

### Steps:

1. **Log in to Dremio**: If you're not already logged in, go to `http://localhost:9047` and log in.

2. **Navigate to Sources**:
   - Click on the **Sources** tab in the left-hand sidebar.

3. **Add MinIO as a Source**:
   - Click **+ Source** in the upper-right corner.
   - Select **Amazon S3** from the list of available source types. MinIO is S3-compatible, so use this option.

4. **Configure MinIO Source**:
   - **Name**: Give your MinIO source a name (e.g., `minio`).
   - **Access Key**: Enter `admin` (as per your Docker Compose environment variables).
   - **Secret Key**: Enter `password` (as per your Docker Compose environment variables).
   - **External Bucket Name**: Specify the bucket you want to access. For example, `datalake`.
   - **Root Path**: Leave blank if accessing the whole bucket, or provide a specific path.
   - **Encryption**: Set this to `None` for unencrypted data. (since your operating in this demo environment)
   - **Enable Compatibility with AWS S3**: Check this option since MinIO is S3-compatible.
   - **Connection Properties**: set the following connection properties:
        - `fs.s3a.path.style.access` to true
        - `fs.s3a.endpoint` to the endpoint of MinIO (minio:9000)

5. **Click Save**: Once the configuration is set, click **Save**. You will now be able to browse and query data stored in MinIO directly from Dremio.

---

## 3. Connecting to Postgres

Dremio natively supports PostgreSQL, making it easy to connect to a Postgres instance and query data from Dremio.

### Steps:

1. **Log in to Dremio**: If you're not already logged in, go to `http://localhost:9047` and log in.

2. **Navigate to Sources**:
   - Click on the **Sources** tab in the left-hand sidebar.

3. **Add Postgres as a Source**:
   - Click **+ Source** in the upper-right corner.
   - Select **PostgreSQL** from the list of available source types.

4. **Configure Postgres Source**:
   - **Name**: Give your Postgres source a name (e.g., `postgres_nessie`).
   - **Hostname**: Enter `postgres` (or `postgres` if referencing the Docker container name).
   - **Port**: Enter `5432`.
   - **Database**: Enter `nessie` (the database name set in your Docker Compose environment variables).
   - **Username**: Enter `nessie` (the Postgres user defined in your Docker Compose environment).
   - **Password**: Enter `nessie` (the Postgres password defined in your Docker Compose environment).

5. **Click Save**: After filling in the details, click **Save**. You can now query data stored in your Postgres database from Dremio.

---

## 4. Connecting to MongoDB

MongoDB is a NoSQL database, and Dremio has native support for MongoDB connections.

### Steps:

1. **Log in to Dremio**: If you're not already logged in, go to `http://localhost:9047` and log in.

2. **Navigate to Sources**:
   - Click on the **Sources** tab in the left-hand sidebar.

3. **Add MongoDB as a Source**:
   - Click **+ Source** in the upper-right corner.
   - Select **MongoDB** from the list of available source types.

4. **Configure MongoDB Source**:
   - **Name**: Give your MongoDB source a name (e.g., `mongo`).
   - **Host**: Enter `mongo` (or `mongo` if referencing the Docker container name).
   - **Port**: Enter `27017` (the default MongoDB port).
   - **Authentication**: If MongoDB is set to require authentication, enable authentication and fill in the following:
     - **Username**: `root` (as per your Docker Compose configuration).
     - **Password**: `example` (as per your Docker Compose configuration).
     - **Authentication Database**: `admin` (the default MongoDB authentication database).
   - **Database**: Specify the default database to connect to (or leave it blank to list all databases).

5. **Click Save**: Once you’ve filled in the necessary details, click **Save**. Dremio will now be connected to MongoDB, allowing you to query collections directly from Dremio.

---

## Summary

Once all the services are connected, you can explore data from:

- **Nessie**: Version-controlled datasets and tables.
- **MinIO**: S3-compatible object storage.
- **Postgres**: Relational database queries.
- **MongoDB**: NoSQL document collections.

These sources can be queried, transformed, and visualized in Dremio using SQL queries. You can also join data from multiple sources (e.g., join MongoDB collections with Postgres tables or MinIO object storage data).

## Troubleshooting

- **Connection Issues**: If you experience issues connecting to any source, ensure that the Docker services are running and accessible. Use `docker ps` to verify the status of containers.
- **Logs**: Check the logs of each container if you face issues. For example:
  - `docker logs postgres`
  - `docker logs mongo`
  - `docker logs minio`
- **Firewalls/Networking**: Ensure that your Docker networking allows access to all services from the Dremio container.

# Query to Join Sample Data

```sql
SELECT 
    c.customer_name, 
    c.email, 
    o.order_date, 
    o.amount, 
    p.preference, 
    p.loyalty_status
FROM 
    postgres.public.customers AS c
JOIN 
    postgres.public.orders AS o ON c.customer_id = o.customer_id
JOIN 
    mongo.mydatabase.customer_preferences AS p ON c.email = p.customer_email;
```