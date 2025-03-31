

# Trino 474 Docker Setup with Hive File Metastore & ADLS Gen2

This repository provides a Docker Compose-based deployment of [Trino 474](https://trino.io/) configured with:

- A Hive connector using a file-based metastore (for development/testing)
- Native support for [Azure Data Lake Storage Gen2 (ADLS Gen2)](https://docs.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction)
- Example SQL statements for creating an external schema/table, inserting data, querying data, and dropping the table

> **Note:** The file-based metastore is intended for single-node, development, or testing purposes. For production environments, a dedicated Hive Metastore service is recommended.

## Table of Contents

- [Overview](#overview)
- [Directory Structure](#directory-structure)
- [Configuration](#configuration)
- [Docker Compose Setup](#docker-compose-setup)
- [Usage](#usage)
- [SQL Examples](#sql-examples)
- [Troubleshooting](#troubleshooting)
- [License](#license)

## Overview

This project demonstrates how to run Trino 474 in a Docker container with:

- **Hive Connector:** Uses a file-based metastore to store table metadata locally.
- **Azure Data Lake Gen2 Support:** Configured with native ADLS Gen2 support using an access key.
- **SQL Examples:** Includes SQL commands to create a schema, create an external table, insert data, query data, and drop the table.

The setup is optimized for a single-node deployment on an 8 GB machine (e.g., DS2_v4 VM). It leverages Docker Compose to mount configuration and persistent data directories.

## Directory Structure
```
trino-setup/
├── docker-compose.yml          # Docker Compose file to run Trino 474
├── etc/                        # Trino configuration directory (mounted to /etc/trino)
│   ├── node.properties         # Node identification and data directory
│   ├── jvm.config              # JVM options (memory, GC settings, etc.)
│   ├── config.properties       # Trino server configuration (coordinator, port, etc.)
│   └── catalog/
│       └── hive.properties     # Hive connector configuration with file metastore & ADLS Gen2 settings
└── data/                       # Persistent data directory (mounted to /data)
    ├── hive-metastore/         # File-based Hive metastore files
    ├── hive-warehouse/         # (Optional) Warehouse directory if used
    └── tmp/                    # Temporary staging directory for write operations
```

## Configuration

### 1. `etc/node.properties`

Identifies the node and sets the data directory.

```properties
node.environment=production
node.id=trino-1
node.data-dir=/data/trino
```

### 2. `etc/jvm.config`

Example JVM settings for an 8 GB machine:

```properties
-server
-Xmx6G
-XX:+UseG1GC
-XX:G1HeapRegionSize=32M
-XX:+ExplicitGCInvokesConcurrent
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow
-XX:ReservedCodeCacheSize=512M
```

### 3. `etc/config.properties`

Sets up the node as a coordinator (and worker) and configures the discovery service.

```properties
coordinator=true
node-scheduler.include-coordinator=true
http-server.http.port=8080
discovery-server.enabled=true
discovery.uri=http://localhost:8080
```

### 4. `etc/catalog/hive.properties`

Configures the Hive connector with a file-based metastore and native ADLS Gen2 support. In Trino 474 you must use a proper URI scheme for local paths and enable the Hadoop filesystem to support file URIs.

```properties
connector.name=hive
hive.metastore=file
hive.metastore.catalog.dir=file:///data/hive-metastore/
hive.temporary-staging-directory-path=file:///data/tmp/

# Enable ADLS Gen2 native support
fs.native-azure.enabled=true
azure.auth-type=ACCESS_KEY
azure.access-key=<YOUR_ADLS_ACCESS_KEY>

# Enable Hadoop FileSystem support for local file URIs
fs.hadoop.enabled=true
```

> **Replace `<YOUR_ADLS_ACCESS_KEY>` with your actual Azure Storage access key.**

## Docker Compose Setup

The `docker-compose.yml` file starts the Trino container and mounts the configuration and data directories:

```yaml
version: '3.8'
services:
  trino:
    image: trinodb/trino:474
    container_name: trino
    user: "1000:1000"  # Run as 'trino' user (UID 1000)
    ports:
      - "8080:8080"
    volumes:
      - ./etc:/etc/trino
      - ./data:/data
    restart: unless-stopped
```

## Usage

1. **Start the Service**

   From the repository root, run:
   ```bash
   docker-compose up -d
   ```
   This launches the Trino server on port `8080`.

2. **Access the Trino CLI**

   You can open an interactive shell inside the container:
   ```bash
   docker exec -it trino trino
   ```
   Alternatively, access the Trino web UI at [http://localhost:8080](http://localhost:8080).

## SQL Examples

### Creating a Schema

Create a schema in the Hive catalog:
```sql
CREATE SCHEMA hive.mydata;
```

### Creating an External Table

Create an external table with partitioning. The table points to Parquet files stored on ADLS Gen2.
```sql
CREATE TABLE hive.mydata.dataset_type_table (
    id INT,
    dataset_type VARCHAR,
    experiment_id VARCHAR -- partition column
)
WITH (
    format = 'PARQUET',
    external_location = 'abfss://results-trino-test@staixpertdestinationdev.dfs.core.windows.net/dataset_type_table/',
    partitioned_by = ARRAY['experiment_id']
);
```

### Inserting Data

*Note: To insert into an external table, ensure you have enabled writes by adding the following property to your `hive.properties`:*
```properties
hive.non-managed-table-writes-enabled=true
```

Then, use an INSERT statement:
```sql
INSERT INTO hive.mydata.dataset_type_table (id, dataset_type, experiment_id)
VALUES 
  (1, 'Type A', 'exp1'),
  (2, 'Type B', 'exp2'),
  (3, 'Type C', 'exp1');
```

### Querying Data

Query the table:
```sql
SELECT * FROM hive.mydata.dataset_type_table LIMIT 10;
```

### Dropping the Table

Drop the table when no longer needed:
```sql
DROP TABLE hive.mydata.dataset_type_table;
```

> **Important:** Since this is an external table, the underlying Parquet files in ADLS Gen2 will not be deleted automatically.

## Troubleshooting

- **File Metastore Errors:**  
  If you encounter errors like `No factory for location`, ensure:
  - You’re using a proper URI scheme (e.g., `file:///`) for local directories.
  - You have enabled Hadoop filesystem support with `fs.hadoop.enabled=true`.
  - The mounted directories (`/data/hive-metastore/`, `/data/tmp/`) exist and are writable by the container.

- **ADLS Connectivity:**  
  Verify that the Azure storage account, container, and access key are correct. Ensure your network/firewall settings allow the Trino container to access ADLS Gen2.

- **Inserting into External Tables:**  
  External tables require enabling non-managed writes. Make sure `hive.non-managed-table-writes-enabled=true` is present in `etc/catalog/hive.properties` if you plan to perform INSERT operations.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
```