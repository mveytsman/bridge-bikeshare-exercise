# Part 1

```
WITH station_to_from_count AS (
  SELECT
    from_station_id AS station_id,
    from_station_name AS station_name,
    COUNT(from_station_name) AS count
  FROM trips
  GROUP BY from_station_id, from_station_name
  UNION
  SELECT
    to_station_id AS station_id,
    to_station_name AS station_name,
    COUNT(to_station_name) AS count
  FROM trips
  GROUP BY to_station_id, to_station_name
), station_count AS (
  SELECT
    station_id,
    station_name,
    SUM(count)
  FROM station_to_from_count
  GROUP BY station_id, station_name
), popular_station_names_and_ids AS (
  SELECT
    station_count.station_id,
    station_count.station_name,
    station_count.sum
  FROM station_count
  JOIN
  (
    SELECT
      MAX(sum),
      station_count.station_id
    FROM station_count
    GROUP BY station_count.station_id
  ) AS max_count
  ON max_count.station_id=station_count.station_id
  WHERE max_count.max=station_count.sum
), null_count AS (
  SELECT *
  FROM station_count
  WHERE (station_count.station_id IS NULL) AND (sum > 3)
), levenshtein_names AS (
  SELECT DISTINCT
    null_count.station_name,
    stations.name,
    levenshtein(stations.name, null_count.station_name)
  FROM null_count
  LEFT JOIN stations
  ON levenshtein(stations.name, null_count.station_name) <= 4
)
```
This `WITH` creates 5 tables.

Tables | Description
--- | ---
`station_to_from_count` | groups station names by column and counts their frequencies; ids are `null` when name doesn't match station names with ids
`station_count` | sums start and points to stations with same name
`popular_station_names_and_ids` | picks station names with largest sum
`null_count` | list station names with `null` for id
`levenshtein_names` | matches station names with `null` for id to popular station names as long the levenshtein is 4 or under

## Question 1
```
INSERT INTO stations
SELECT station_id, station_name
FROM popular_station_names_and_ids;
```
This will populate the `stations` table with ids and names.

## Question 3
```
UPDATE trips
SET to_station_name=stations.name
FROM stations
WHERE trips.to_station_id=stations.id
```
```
UPDATE trips
SET from_station_name=stations.name
FROM stations
WHERE trips.from_station_id=stations.id
```
Now make the stations with the same id have the same name in trips table.

```
UPDATE trips
SET to_station_id = stations.id
FROM stations
WHERE (name = trips.to_station_name) AND (trips.to_station_id IS NULL);
```
```
UPDATE trips
SET from_station_id = stations.id
FROM stations
WHERE name = (trips.from_station_name) AND (trips.from_station_id IS NULL);
```
Where station id is null and station name is consistent, populate id.

```
UPDATE trips
SET to_station_name = stations.name
FROM stations
WHERE (levenshtein(stations.name, trips.to_station_name) <= 4) AND (trips.to_station_id IS NULL);
```
```
UPDATE trips
SET from_station_name = stations.name
FROM stations
WHERE (levenshtein(stations.name, trips.from_station_name) <= 4) AND (trips.from_station_id IS NULL);
```
Replace names with suitable matches -- levenshtein of 4 is good, whereas 6 is not good.
There are a handful of stations that don't have a good match...what to do with them?

# Part 2
## Question 1

```
SELECT
 mode() WITHIN GROUP (ORDER BY start_time_str),
 original_filename
FROM trips
GROUP BY original_filename;
```
This will sample one `start_time_str` per quarter.

Quarter | String formats
--- | ---
2017 Q1 | dd/mm/yyyy HH:MM
2017 Q2 | dd/mm/yyyy HH:MM
2017 Q3 | mm/dd/yyyy HH:MM
2017 Q4 | mm/dd/yyyy HH:MM:SS
2018 Q1 | mm/dd/yyyy HH:MM
2018 Q2 | mm/dd/yyyy HH:MM
2018 Q3 | mm/dd/yyyy HH:MM
2018 Q4 | mm/dd/yyyy HH:MM

## Question 2

```
UPDATE trips
SET start_time=TO_TIMESTAMP(start_time_str, 'DD/MM/YYYY HH24:MI')
WHERE original_filename='Bikeshare Ridership (2017 Q1).csv' OR original_filename='Bikeshare Ridership (2017 Q2).csv';
```
```
UPDATE trips
SET end_time=TO_TIMESTAMP(end_time_str, 'DD/MM/YYYY HH24:MI')
WHERE original_filename='Bikeshare Ridership (2017 Q1).csv' OR original_filename='Bikeshare Ridership (2017 Q2).csv';
```
```
UPDATE trips
SET start_time=TO_TIMESTAMP(start_time_str, 'MM/DD/YYYY HH24:MI:SS')
WHERE original_filename='Bikeshare Ridership (2017 Q4).csv';
```
```
UPDATE trips
SET end_time=TO_TIMESTAMP(end_time_str, 'MM/DD/YYYY HH24:MI:SS')
WHERE (original_filename='Bikeshare Ridership (2017 Q4).csv') AND (id != 2302635);
```
```
UPDATE trips
SET start_time=TO_TIMESTAMP(start_time_str, 'MM/DD/YYYY HH24:MI')
WHERE start_time IS NULL;
```
```
UPDATE trips
SET end_time=TO_TIMESTAMP(end_time_str, 'MM/DD/YYYY HH24:MI')
WHERE end_time IS NULL AND (id != 2302635);
```
Ride entry with `id` 2302635 has `NULLNULL` for a value, which I ignore

# Part 3
## Question 1
```
SELECT
  EXTRACT(MONTH FROM start_time),
  COUNT(*)
FROM trips
GROUP BY EXTRACT(MONTH FROM start_time);
```
This will look at cumulative seasonal trends. Also, trips are counted when they start.

## Question 2

```
WITH rides_by_hour AS (
  SELECT
    EXTRACT(HOUR FROM start_time) AS start_time,
    EXTRACT(HOUR FROM end_time) AS end_time
  FROM trips
)

SELECT
  start_time,
  COUNT(start_time),
  COUNT(end_time)
FROM rides_by_hour
GROUP BY start_time
ORDER BY start_time;
```

## Question 3

```
SELECT
  from_station_name,
  count(from_station_id)
FROM trips
WHERE
  EXTRACT(MONTH FROM start_time) = 6 OR
  EXTRACT(MONTH FROM start_time) = 7 OR
  EXTRACT(MONTH FROM start_time) = 8
GROUP BY from_station_name
ORDER BY count DESC
LIMIT 10;
```

## Question 4
```
SELECT
  from_station_name,
  count(from_station_id)
FROM trips
WHERE
  EXTRACT(MONTH FROM start_time) = 12 OR
  EXTRACT(MONTH FROM start_time) = 1 OR
  EXTRACT(MONTH FROM start_time) = 2
GROUP BY from_station_name
ORDER BY count DESC
LIMIT 10;
```

## Question 5

There are other ways to check the integrity of the data.
For example, there are over 1,000 trips that last less than 10 seconds.
```
SELECT *
FROM trips
WHERE (to_station_id IS NULL AND from_station_id IS NULL) AND (duration_seconds < 10)
ORDER BY duration_seconds ASC;
```
In particular, when the station id's are null, trip duration produces
interesting results
```
SELECT MAX(duration_seconds), MIN(duration_seconds), stddev(duration_seconds), avg(duration_seconds)
FROM trips
WHERE (to_station_id IS NULL AND from_station_id IS NULL);
```

 row | max | min | stddev | avg
 --- | --- | --- | --- | ---
 1 | 6382030 | 0 | 11757.7... | 1082.5...

 Whereas trips with existing station id's have slightly better values.
 ```
 SELECT MAX(duration_seconds), MIN(duration_seconds), stddev(duration_seconds), avg(duration_seconds)
FROM trips
WHERE (to_station_id IS NOT NULL AND from_station_id IS NOT NULL)
 ```
 row | max | min | stddev | avg
 --- | --- | --- | --- | ---
 1 | 550777 | 60 | 1510.9... | 945.7...

What could have happened to the data that changes the quality so much?
