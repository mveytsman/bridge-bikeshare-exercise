
# Homework

## Instructions 

Clone this repo and make a PR with your answers to the questions below, and tag your reviewer in the PR.

Keep good notes of the queries you run.

## Part 1: Missing Station Data

As we've discussed in class, some of the station_id columns are NULL because of missing data. We also have inconsistent names for some of the station IDs.

Your database contains a `stations` schema:

```sql
CREATE TABLE stations
(
  id INT NOT NULL PRIMARY KEY,
  name VARCHAR(100) NOT NULL
);
```

1) Write a query that fills out this table. Try your best to pick the correct station name for each ID. You may have to make some manual choices or editing based on the inconsistencies we've found. Do try to pick the correct name for each station ID based on how popular it is in the trip data.

Hint: your query will look something like 

``` sql
INSERT INTO stations (SELECT ... FROM tripe);
```

Hint 2: You don't have to do it all in one query

---

First add some indexes

``` sql
CREATE INDEX trips_from_station_id_idx ON public.trips USING btree (from_station_id)
CREATE INDEX trips_to_station_id_idx ON public.trips USING btree (to_station_id)
```

Next lets get all the stations 

``` sql
SELECT from_station_id, array_agg(distinct from_station_name) FROM 
FROM trips 
WHERE from_station_id IS NOT null
GROUP BY from_station_id
LIMIT 10
```


Let's find the mislabeled data

``` sql
SELECT from_station_id, array_agg(distinct from_station_name), levenshtein((array_agg(distinct from_station_name))[1],(array_agg(distinct from_station_name))[2])
FROM trips
WHERE from_station_id IS NOT null
GROUP BY from_station_id
HAVING count(distinct from_station_name) > 1
LIMIT 10
```

Set the levenstein distance higher to see a set of possibly bad ones

``` sql
SELECT from_station_id, array_agg(distinct from_station_name), levenshtein((array_agg(distinct from_station_name))[1],(array_agg(distinct from_station_name))[2])
FROM trips
WHERE from_station_id IS NOT null
GROUP BY from_station_id
HAVING count(distinct from_station_name) > 1
AND  levenshtein((array_agg(distinct station_name))[1],(array_agg(distinct station_name))[2]) > 20
```

Bad ids 7029, 7084, 7086


Picking the first station

``` sql
SELECT from_station_id, (array_agg(distinct from_station_name))[1]
FROM trips
WHERE from_station_id IS NOT null
AND from_station_id NOT IN (7029, 7084, 7086)
GROUP BY from_station_id
```

(can also do this with DISTINCT ON, see https://zaiste.net/postgresql_distinct_on/)

Picking the most popular station

I didn't know this, by postgres has a MDOE. It looks like this

``` sql
SELECT from_station_id, mode() WITHIN GROUP (ORDER BY from_station_name) as from_station_name
FROM trips
WHERE from_station_id IS NOT null
AND from_station_id NOT IN (7029, 7084, 7086)
GROUP BY from_station_id
```

Let's do it by hand though!

First, we want the count of the station names for each station id. We can think of this as counting the pairs

``` sql
SELECT from_station_id, from_station_name, count(*) 
FROM trips
GROUP BY from_station_id, from_station_name
```

We can save this as a CTE for later:


``` sql
WITH pair_counts AS (
  SELECT from_station_id, from_station_name, count(*) 
  FROM trips
  GROUP BY from_station_id, from_station_name
)
```

We can then see the order for a given for stations: 

``` sql
SELECT *, ROW_NUMBER() OVER (
  PARTITION BY from_station_id ORDER BY count
)
FROM pair_counts
```


We cant use the row number in a where without a CTE so let's do that again.
Putting it together: 

``` sql
WITH pair_counts AS (
  SELECT from_station_id, from_station_name, count(*) 
  FROM trips
  GROUP BY from_station_id, from_station_name
),

pair_ordering AS (
  SELECT *, ROW_NUMBER() OVER (
    PARTITION BY from_station_id ORDER BY count
  )
  FROM pair_counts
)


SELECT * FROM pair_ordering
WHERE row_number = 1 
AND from_station_id IS NOT null
AND from_station_id NOT IN (7029, 7084, 7086)
```

As an alternative to the window function, we can use distinct on with an orderin

```sql
SELECT DISTINCT ON(from_station_id) *
FROM pair_counts
WHERE from_station_id IS NOT null
AND from_station_id NOT IN (7029, 7084, 7086)
ORDER BY from_station_id, count
```


Okay, lets fill the stations table:

``` sql
INSERT INTO stations
(SELECT from_station_id, from_station_name FROM pair_ordering
WHERE row_number = 1 
AND from_station_id IS NOT null
AND from_station_id NOT IN (7029, 7084, 7086))
```

TODO the 3 weird rows

---


2) Should we add any indexes to the stations table, why or why not?



---

We might want an index for name for the next step

``` sql
CREATE INDEX stations_name_idx ON stations USING btree (name)

```

---


3) Fill in the missing data in the `trips` table based on the work you did above


---

Let's find the station names that match:

``` sql

SELECT count(distinct from_station_name) FROM trips 
INNER JOIN stations ON trips.from_station_name = stations.name
WHERE from_station_id IS NULL 

238
```

``` sql
SELECT count(distinct from_station_name) FROM trips 
LEFT JOIN stations ON trips.from_station_name = stations.name
WHERE from_station_id IS NULL 
AND stations.id IS NULL

33
```

Let's fill in the first part


``` sql
UPDATE trips
SET from_station_id = stations.id
FROM stations
WHERE trips.from_station_name = stations.name
AND from_station_id IS NULL 
```


For the second part, let's look at our pair counts again

``` sql
WITH pair_counts AS (
  SELECT from_station_id, from_station_name, count(*) 
  FROM trips
  GROUP BY from_station_id, from_station_name
)

SELECT * FROM trips
INNER JOIN pair_counts ON trips.from_station_name = pair_counts.from_station_name
WHERE trips.from_station_id IS null
AND pair_counts.from_station_id IS NOT NULL
LIMIT 10
```

Now we can do another update

``` sql
UPDATE trips
SET from_station_id = pair_counts.from_station_id
FROM pair_counts
WHERE trips.from_station_name = pair_counts.from_station_name
AND trips.from_station_id IS null
AND pair_counts.from_station_id IS NOT NULL

```

We still have some missing trips :(
``` sql
SELECT count(distinct from_station_name) FROM trips 
LEFT JOIN stations ON trips.from_station_name = stations.name
WHERE from_station_id IS NULL 
AND stations.id IS NULL

12
```

TODO: do the same for to_station_id

---

## Part 2: Missing Date Data

You may have noticed that we have the dates as strings. This is because the dataset we have uses an inconsistent format for dates 😔😔😔

Note that the `original_filename` column is broken down into quarters, knowing this is helpful here!

1) What's the inconsistency in date formats? You can assume that each quarter's trips are numbered sequentially, starting with the first day of the first month of that quarter.


---

First we can get a quick peek at the data

``` sql

SELECT DISTINCT ON(original_filename) start_time_str, original_filename FROM trips

```

NOTE that 2017 Q4 has seconds, and we have some padding issues too

Let's find the ones in Day/Month/Year

``` sql
SELECT DISTINCT ON(original_filename) start_time_str, original_filename FROM trips
WHERE start_time_str LIKE '22/%'


Bikeshare Ridership (2017 Q1).csv
Bikeshare Ridership (2017 Q2).csv
```

Just to make sure, let's see the ones that aren't

``` sql
SELECT DISTINCT ON(original_filename) start_time_str, original_filename FROM trips
WHERE start_time_str LIKE '%/22/%'

Bikeshare Ridership (2017 Q4).csv
Bikeshare Ridership (2017 Q3).csv
Bike Share Toronto Ridership_Q1 2018.csv
Bike Share Toronto Ridership_Q2 2018.csv
Bike Share Toronto Ridership_Q4 2018.csv
Bike Share Toronto Ridership_Q3 2018.csv

```

NOTE THESE ARE not 0 padded!


- Bikeshare Ridership (2017 Q1).csv - DD/MM/YYYY HH24:MI
- Bikeshare Ridership (2017 Q2).csv - DD/MM/YYYY HH24:MI
- Bikeshare Ridership (2017 Q3).csv - FMMM/DD/YYYY HH24:MI
- Bikeshare Ridership (2017 Q4).csv - FMMM/DD/YYYY HH24:MI:SS
- Bike Share Toronto Ridership_Q1 2018.csv - FMMM/DD/YYYY HH24:MI
- Bike Share Toronto Ridership_Q2 2018.csv - FMMM/DD/YYYY HH24:MI
- Bike Share Toronto Ridership_Q3 2018.csv - FMMM/DD/YYYY HH24:MI
- Bike Share Toronto Ridership_Q4 2018.csv - FMMM/DD/YYYY HH24:MI


---

2) Take a look at Postgres's [date functions](https://www.postgresql.org/docs/12/functions-datetime.html), and fill in the missing date data using proper timestamps. You may have to write several queries to do this.

Hint: your queries will look something like

``` sql
UPDATE trips
SET start_time = ..., end_time = ...
WHERE ...;
```

---

This works

``` sql
SELECT 
CASE 
  WHEN original_filename = 'Bikeshare Ridership (2017 Q1).csv' OR original_filename = 'Bikeshare Ridership (2017 Q2).csv'
  THEN 
    to_timestamp(start_time_str, 'DD/MM/YYYY HH24:MI') 
  WHEN original_filename = 'Bikeshare Ridership (2017 Q4).csv'
  THEN
    to_timestamp(start_time_str, 'FMMM/DD/YYYY HH24:MI:SS') 
  ELSE 
    to_timestamp(start_time_str, 'FMMM/DD/YYYY HH24:MI') 
  END
FROM trips LIMIT 10;

```

Putting it together

``` sql
UPDATE trips
SET start_time = 
CASE 
WHEN original_filename = 'Bikeshare Ridership (2017 Q1).csv' OR original_filename = 'Bikeshare Ridership (2017 Q2).csv'
THEN 
  to_timestamp(start_time_str, 'DD/MM/YYYY HH24:MI') 
WHEN original_filename = 'Bikeshare Ridership (2017 Q4).csv'
THEN
    to_timestamp(start_time_str, 'FMMM/DD/YYYY HH24:MI:SS')
ELSE 
  to_timestamp(start_time_str, 'FMMM/DD/YYYY HH24:MI') 
END,
end_time = 
CASE 
WHEN original_filename = 'Bikeshare Ridership (2017 Q1).csv' OR original_filename = 'Bikeshare Ridership (2017 Q2).csv'
THEN
  to_timestamp(end_time_str, 'DD/MM/YYYY HH24:MI') 
WHEN original_filename = 'Bikeshare Ridership (2017 Q4).csv'
THEN
    to_timestamp(end_time_str, 'FMMM/DD/YYYY HH24:MI:SS')
ELSE 
  to_timestamp(end_time_str, 'FMMM/DD/YYYY HH24:MI') 
END


```

---

3) Other than the index in class, would we benefit from any other indexes on this table? Why or why not?

## Part 3: Data-driven insights

Using the table you made in part 1 and the dates you added in part 2, let's answer some questions about the bike share data

1) Build a mini-report that does a breakdown of number of trips by month
2) Build a mini-report that does a breakdown of number trips by time of day of their start and end times
3) What are the most popular stations to bike to in the summer?
4) What are the most popular stations to bike from in the winter?
5) Come up with a question that's interesting to you about this data that hasn't been asked and answer it.


