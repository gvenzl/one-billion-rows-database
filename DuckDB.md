# Set timing on

```sql
.timer on
```

# Query external file

```sql
SELECT *
 FROM read_csv('/opt/obrc_ramfs/measurements_413.txt')
 LIMIT 10;
```

# Scan 10 billion rows - parallel

```sql
SET threads=128;
SELECT COUNT(*)
 FROM read_csv('/opt/obrc_ramfs/measurements_413.txt');
```

# Aggregate 10 billion rows

```sql
SELECT station_name,
       MIN(measurement) AS min_measurement,
       ROUND(AVG(measurement),1) AS mean_measurement,
       MAX(measurement) AS max_measurement
 FROM read_csv('/opt/obrc_ramfs/measurements_413.txt',
               header=false,
               delim=';',
               columns=
               {
                'station_name':'VARCHAR',
                'measurement':'FLOAT'
               }
              )
 GROUP BY station_name
 ORDER BY station_name;
```

# Aggregate 10 billion rows into one row return

```sql
SELECT ARRAY_TO_STRING(
        LIST_SORT(
         LIST(
          station_name || '=' ||
           CONCAT_WS('/', min_measurement, mean_measurement, max_measurement)
         )
        ),
        ', '
       ) AS result
 FROM (
  SELECT station_name,
         MIN(measurement) AS min_measurement,
         ROUND(AVG(measurement),1) AS mean_measurement,
         MAX(measurement) AS max_measurement
   FROM read_csv('/opt/obrc_ramfs/measurements_413.txt',
                 header=false,
                 delim=';',
                 columns=
                 {
                  'station_name':'VARCHAR',
                  'measurement':'FLOAT'
                 }
                )
   GROUP BY station_name
 );
```

# Load data into transient database table

```sql
CREATE TABLE obrc
 AS
  SELECT *
   FROM read_csv('/opt/obrc_ramfs/measurements_413.txt',
                 header=false,
                 delim=';',
                 columns=
                 {
                  'station_name':'VARCHAR',
                  'measurement':'FLOAT'
                 }
                );
```

# Aggregate 10 billion rows from internal table

```sql
SELECT station_name,
       MIN(measurement) AS min_measurement,
       ROUND(AVG(measurement),1) AS mean_measurement,
       MAX(measurement) AS max_measurement
 FROM obrc
 GROUP BY station_name
 ORDER BY station_name;
```

# Aggregate 10 billion rows into one row return from internal table

```sql
SELECT ARRAY_TO_STRING(
        LIST_SORT(
         LIST(
          station_name || '=' ||
           CONCAT_WS('/', min_measurement, mean_measurement, max_measurement)
         )
        ),
        ', '
       ) AS result
 FROM (
  SELECT station_name,
         MIN(measurement) AS min_measurement,
         ROUND(AVG(measurement),1) AS mean_measurement,
         MAX(measurement) AS max_measurement
   FROM obrc
   GROUP BY station_name
 );
```
