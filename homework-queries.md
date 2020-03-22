Part 1


WITH station_to_from_count AS (
  SELECT
    max(from_station_id) AS station_id,
    from_station_name AS station_name,
    count(from_station_name) AS count
  FROM trips
  GROUP BY from_station_name
  UNION
  SELECT
    max(to_station_id) AS station_id,
    to_station_name AS station_name,
    count(to_station_name) AS count
  FROM trips
  GROUP BY to_station_name
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
  SELECT DISTINCT null_count.station_name, stations.name, levenshtein(stations.name, null_count.station_name)
  FROM null_count
  LEFT JOIN stations
  ON levenshtein(stations.name, null_count.station_name) <= 4
)

SELECT *
FROM null_count
JOIN popular_station_names_and_ids
ON null_count.station_name=popular_station_names_and_ids.station_name;

INSERT INTO stations
SELECT station_id, station_name
FROM popular_station_names_and_ids;

/*
  matches station names to station ids by popularity
  delete station_name with values of "Base Station" and "NULL"
  only 11 stations that need manual input
  no station names associated with null ids match with popular station names
*/

/*
  Now make the stations with the same id have the same name in trips table
*/

UPDATE trips
SET to_station_name=stations.name
FROM stations
WHERE trips.to_station_id=stations.id

UPDATE trips
SET from_station_name=stations.name
FROM stations
WHERE trips.from_station_id=stations.id

/*
  time to add id's to stations with null id's
*/

/*
  fill id's with matching names
*/

UPDATE trips
SET to_station_id = stations.id
FROM stations
WHERE (name = trips.to_station_name) AND (trips.to_station_id IS NULL);

UPDATE trips
SET from_station_id = stations.id
FROM stations
WHERE name = (trips.from_station_name) AND (trips.from_station_id IS NULL);

SELECT *
FROM null_count
JOIN popular_station_names_and_ids
ON popular_station_names_and_ids.station_name = null_count.station_name;
/*
  sanity check: no exact matches for items in null_count, must do manually
*/

Select name
FROM stations
WHERE name LIKE '%Bathurst%Queens Quay%'

/*
  Rows Affected: 0
*/

UPDATE trips
SET to_station_name=(
  Select name
  FROM stations
  WHERE name LIKE '%Bathurst%Queens Quay%'
)
FROM stations
WHERE trips.to_station_name='Bathurst St / Queens Quay W'

UPDATE trips
SET from_station_name=(
  Select name
  FROM stations
  WHERE name LIKE '%Bathurst%Queens Quay%'
)
FROM stations
WHERE trips.from_station_name='Bathurst St / Queens Quay W'

SELECT name
FROM stations
WHERE levenshtein('Beverly St / College St', name) <= 4;

UPDATE trips
SET to_station_name = (
  SELECT name
  FROM stations
  WHERE levenshtein('Beverly St / College St', name) <= 4
)
FROM stations
WHERE trips.to_station_name = 'Beverly St / College St';

UPDATE trips
SET from_station_name = (
  SELECT name
  FROM stations
  WHERE levenshtein('Beverly St / College St', name) <= 4
)
FROM stations
WHERE trips.from_station_name = 'Beverly St / College St';

/*
  more programmatically
*/

SELECT
  station_name,
  name,
  levenshtein
FROM
  levenshtein_names
ORDER BY
 station_name;

/*
  replace names with suitable matches -- levenshtein of 4 is good, whereas 6 is not good
  also, the values are unique
*/

UPDATE trips
SET to_station_name = stations.name
FROM stations
WHERE (levenshtein(stations.name, trips.to_station_name) <= 4) AND (trips.to_station_id IS NULL);

UPDATE trips
SET from_station_name = stations.name
FROM stations
WHERE (levenshtein(stations.name, trips.from_station_name) <= 4) AND (trips.from_station_id IS NULL);

UPDATE trips
SET to_station_name=(
  Select name
  FROM stations
  WHERE name LIKE '%Bathurst%Queens Quay%'
)
FROM stations
WHERE trips.to_station_name='Bathurst St / Queens Quay W'

/*
  There are a handful of stations that don't have a good match...what to do with them?
*/

Part 2


/*
  pick one time per quarter
  Bikeshare Ridership (2017 Q1).csv
  Bike Share Toronto Ridership_Q1 2018.csv
*/

SELECT
 mode() WITHIN GROUP (ORDER BY start_time_str),
 original_filename
FROM trips
GROUP BY original_filename;

2017
Q1 - dd/mm/yyyy HH:MM
Q2 - dd/mm/yyyy HH:MM
Q3 - mm/dd/yyyy HH:MM
Q4 - mm/dd/yyyy HH:MM:SS
2018
Q1 - mm/dd/yyyy HH:MM
Q2 - mm/dd/yyyy HH:MM
Q3 - mm/dd/yyyy HH:MM
Q4 - mm/dd/yyyy HH:MM

UPDATE trips
SET start_time=TO_TIMESTAMP(start_time_str, 'DD/MM/YYYY HH24:MI')
WHERE original_filename='Bikeshare Ridership (2017 Q1).csv' OR original_filename='Bikeshare Ridership (2017 Q2).csv';

UPDATE trips
SET end_time=TO_TIMESTAMP(end_time_str, 'DD/MM/YYYY HH24:MI')
WHERE original_filename='Bikeshare Ridership (2017 Q1).csv' OR original_filename='Bikeshare Ridership (2017 Q2).csv';

UPDATE trips
SET start_time=TO_TIMESTAMP(start_time_str, 'MM/DD/YYYY HH24:MI:SS')
WHERE original_filename='Bikeshare Ridership (2017 Q4).csv';

UPDATE trips
SET end_time=TO_TIMESTAMP(end_time_str, 'MM/DD/YYYY HH24:MI:SS')
WHERE (original_filename='Bikeshare Ridership (2017 Q4).csv') AND (id != 2302635);

UPDATE trips
SET start_time=TO_TIMESTAMP(start_time_str, 'MM/DD/YYYY HH24:MI')
WHERE start_time IS NULL;

UPDATE trips
SET end_time=TO_TIMESTAMP(end_time_str, 'MM/DD/YYYY HH24:MI')
WHERE end_time IS NULL AND (id != 2302635);
