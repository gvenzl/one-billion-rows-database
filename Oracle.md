# Enable Parallel Workers

```sql
conn / as sysdba
ALTER SYSTEM SET parallel_max_servers=128;
ALTER SYSTEM SET parallel_servers_target=128;
ALTER SYSTEM SET parallel_min_servers=128;
```

# Use Direct I/O

```sql
ALTER SYSTEM SET filesystemio_options=DIRECTIO SCOPE=SPFILE;
```

# Use 500GB RAM

```sql
ALTER SYSTEM SET sga_max_size         =300g SCOPE=SPFILE;
ALTER SYSTEM SET sga_target           =300g SCOPE=SPFILE;
ALTER SYSTEM SET pga_aggregate_target =100g;
ALTER SYSTEM SET pga_aggregate_limit  =200g;
```

# Restart database

```sql
SHUTDOWN IMMEDIATE;
STARTUP;
```

# Create tablespace

```sql
ALTER SESSION SET CONTAINER=ORCLPDB1;
CREATE BIGFILE TABLESPACE obrc_ts
 DATAFILE '/opt/oracle/oradata/ORCLCDB/ORCLPDB1/obrc_ts.dbf'
 SIZE 30g
 AUTOEXTEND ON
  NEXT 1g
  MAXSIZE UNLIMITED;
```

# Create filesystem directory

```sql
CREATE DIRECTORY obrc AS '/opt/obrc/';
CREATE DIRECTORY obrc_ramfs AS '/opt/obrc_ramfs/';
```

# Create user and grant privileges

```sql
CREATE USER obrc IDENTIFIED BY obrc DEFAULT TABLESPACE obrc_ts QUOTA UNLIMITED ON obrc_ts;

GRANT CONNECT, RESOURCE TO obrc;

GRANT READ ON DIRECTORY obrc TO obrc;
GRANT READ ON DIRECTORY obrc_ramfs TO obrc;
```

# Connect as obrc

```sql
CONNECT obrc/obrc@localhost/orclpdb1
```

# Set timing on

```sql
SET TIMING ON;
```

# Query external file

```sql
SELECT *
 FROM EXTERNAL
 (
   (
     station_name  VARCHAR2(26),
     measurement   NUMBER(3,1)
   )
   TYPE oracle_loader
   DEFAULT DIRECTORY obrc_ramfs
   ACCESS PARAMETERS
   (
     RECORDS DELIMITED BY NEWLINE
     NOBADFILE
     NOLOGFILE
     FIELDS TERMINATED BY ';'
     (
       station_name CHAR(26),
       measurement  CHAR(5)
     )
   )
   LOCATION ('measurements_413.txt')
   REJECT LIMIT UNLIMITED
 ) measurements
FETCH FIRST 10 ROWS ONLY;
```

# Create external tables

```sql
CREATE TABLE obrc_ext
(
  station_name VARCHAR2(26),
  measurement  NUMBER(3,1)
)
ORGANIZATION EXTERNAL
(
  TYPE oracle_loader
  DEFAULT DIRECTORY obrc
  ACCESS PARAMETERS
  (
    RECORDS DELIMITED BY NEWLINE
    NOBADFILE
    NOLOGFILE
    FIELDS TERMINATED BY ';'
    (
      station_name CHAR(26),
      measurement  CHAR(5)
    )
  )
  LOCATION ('measurements_413.txt')
)
REJECT LIMIT UNLIMITED;

CREATE TABLE obrc_ramfs_ext
(
  station_name VARCHAR2(26),
  measurement  NUMBER(3,1)
)
ORGANIZATION EXTERNAL
(
  TYPE oracle_loader
  DEFAULT DIRECTORY obrc_ramfs
  ACCESS PARAMETERS
  (
    RECORDS DELIMITED BY NEWLINE
    NOBADFILE
    NOLOGFILE
    FIELDS TERMINATED BY ';'
    (
      station_name CHAR(26),
      measurement  CHAR(5)
    )
  )
  LOCATION ('measurements_413.txt')
)
REJECT LIMIT UNLIMITED;
```

# Scan 10 billion rows - parallel

```sql
SELECT /*+ PARALLEL (128) */
 COUNT(*)
 FROM obrc_ramfs_ext;
```

# Aggregate 10 billion rows

```sql
SELECT /*+ PARALLEL (128) */
  station_name,
  MIN(measurement) AS min_measurement,
  ROUND(AVG(measurement),1) AS mean_measurement,
  MAX(measurement) AS max_measurement
 FROM obrc_ramfs_ext
 GROUP BY station_name;
```

# Aggregate 10 billion rows into one row return

```sql
SELECT /*+ PARALLEL (128) */
  '{' ||
    LISTAGG(station_name || '=' || min_measurement || '/' || mean_measurement || '/' || max_measurement, ', ')
      WITHIN GROUP (ORDER BY station_name) ||
  '}' AS result
 FROM
  (SELECT station_name,
          MIN(measurement) AS min_measurement,
          ROUND(AVG(measurement), 1) AS mean_measurement,
          MAX(measurement) AS max_measurement
    FROM obrc_ramfs_ext
     GROUP BY station_name
  );
```

# Load data into database table

```sql
CREATE TABLE obrc
 PARALLEL 128
 AS
  SELECT *
   FROM obrc_ramfs_ext;
```

# Aggregate 10 billion rows from internal table

```sql
SELECT /*+ PARALLEL (128) */
  station_name,
  MIN(measurement) AS min_measurement,
  ROUND(AVG(measurement),1) AS mean_measurement,
  MAX(measurement) AS max_measurement
 FROM obrc
 GROUP BY station_name;
```

# Aggregate 10 billion rows into one row return from internal table

```sql
SELECT /*+ PARALLEL (128) */
  '{' ||
    LISTAGG(station_name || '=' || min_measurement || '/' || mean_measurement || '/' || max_measurement, ', ')
      WITHIN GROUP (ORDER BY station_name) ||
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
DROP TABLE obrc purge;
DROP TABLE obrc_ext;
DROP TABLE obrc_ramfs_ext;

conn / as sysdba

ALTER SESSION SET CONTAINER=ORCLPDB1;

DROP USER obrc;

DROP DIRECTORY obrc;
DROP DIRECTORY obrc_ramfs;

DROP TABLESPACE obrc_ts INCLUDING CONTENTS AND DATAFILES;
```
