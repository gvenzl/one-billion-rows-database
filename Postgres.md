# Install additional modules

```console
sudo dnf install postgresql17-contrib
```

# Enable Parallel Workers

```sql
psql
ALTER SYSTEM SET max_worker_processes=128;
ALTER SYSTEM SET max_parallel_workers=128;
ALTER SYSTEM SET max_parallel_workers_per_gather=128;
ALTER SYSTEM SET autovacuum_max_workers=128;
```

# Use 500GB RAM

```sql
ALTER SYSTEM SET shared_buffers='300GB';
ALTER SYSTEM SET work_mem='10GB';
exit;
```

# Restart database

```console
sudo systemctl restart postgresql-17
```

# Create database

```sql
psql
CREATE DATABASE obrc;
\c obrc;
```

# Create foreign data wrapper and foreign data server

```sql
CREATE EXTENSION file_fdw;
CREATE SERVER csv_data FOREIGN DATA WRAPPER file_fdw;
```

# Create user and grant privileges

```sql
CREATE USER obrc PASSWORD 'obrc';
CREATE SCHEMA AUTHORIZATION obrc;
GRANT ALL PRIVILEGES ON DATABASE obrc TO obrc;
GRANT USAGE ON FOREIGN SERVER csv_data TO obrc;
GRANT pg_read_server_files TO obrc;
exit;
```

# Connect as obrc

```sql
psql -U obrc -d obrc -W -h localhost
```

# Create external tables

```sql
CREATE FOREIGN TABLE obrc_ext
(
  station_name VARCHAR(26),
  measurement  NUMERIC(3,1)
)
SERVER csv_data
  OPTIONS
  (
    filename '/opt/obrc/measurements_413.txt',
    format 'csv',
    delimiter ';'
  );

CREATE FOREIGN TABLE obrc_ramfs_ext
(
  station_name VARCHAR(26),
  measurement  NUMERIC(3,1)
)
SERVER csv_data
  OPTIONS
  (
    filename '/opt/obrc_ramfs/measurements_413.txt',
    format 'csv',
    delimiter ';'
  );
```

# Set timing on

```sql
\timing
```

# Query external file

```sql
SELECT *
 FROM obrc_ramfs_ext
 FETCH FIRST 10 ROWS ONLY;
```

# Scan 10 billion rows - no parallel possible

```sql
SELECT
 COUNT(*)
 FROM obrc_ramfs_ext;
```

# Load data into database table

```sql
CREATE UNLOGGED TABLE obrc
(
  station_name VARCHAR(26),
  measurement  NUMERIC(3,1)
);

COPY obrc(station_name, measurement)
 FROM '/opt/obrc_ramfs/measurements_413.txt'
 WITH
 (
  FORMAT CSV,
  DELIMITER ';'
 );
```

# Aggregate 10 billion rows from internal table

```sql
SELECT
  station_name,
  MIN(measurement) AS min_measurement,
  ROUND(AVG(measurement),1) AS mean_measurement,
  MAX(measurement) AS max_measurement
 FROM obrc
 GROUP BY station_name;
```

# Aggregate 10 billion rows into one row return from internal table

```sql
SELECT
  '{' ||
    STRING_AGG(station_name || '=' || min_measurement || '/' || mean_measurement || '/' || max_measurement, ', ' ORDER BY station_name) ||
  '}' AS result
 FROM
  (SELECT station_name,
          MIN(measurement) AS min_measurement,
          ROUND(AVG(measurement), 1) AS mean_measurement,
          MAX(measurement) AS max_measurement
    FROM obrc
     GROUP BY station_name
  );
```

# Cleanup

```sql
psql
DROP DATABASE obrc;
DROP USER obrc;
```
