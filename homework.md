# Homework

## Opening note

This is about data cleanup, and a real world dataset, so **it’s okay to do your best and skip inconsistencies in the data that are hard to fix**. It’s valuable to figure out what the issues in the data are and I want you to challenge yourself to write queries that let you answer questions about the data, but **if you’re stuck on a station that doesn’t make you can just throw it away and move on, just note it and only come back later if you have time and want to.**

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
INSERT INTO stations (SELECT ... FROM trips);
```

Hint 2: You don't have to do it all in one query

2) Should we add any indexes to the stations table, why or why not?

3) Fill in the missing data in the `trips` table based on the work you did above

## Part 2: Missing Date Data

You may have noticed that we have the dates as strings. This is because the dataset we have uses an inconsistent format for dates 😔😔😔

Note that the `original_filename` column is broken down into quarters, knowing this is helpful here!

There's an inconsistency in the dates in the string columns. For example trip `4461217` has a `start_time_str` of `11/25/2018 15:47` which implies that it's Month / Day / Year, since there's no 25th month, while trip `928659` has a `start_time_str` of `22/04/2017 0:04` which implies Day / Month / Year, since there's not 22nd month!

We can assume that each `original_filename` has dates in the same format. 

1) What's the inconsistency in date formats? Which files are which?

2) Take a look at Postgres's [date functions](https://www.postgresql.org/docs/12/functions-datetime.html), and fill in the missing date data using proper timestamps. You may have to write several queries to do this.

Hint: your queries will look something like

``` sql
UPDATE trips
SET start_time = ..., end_time = ...
WHERE ...;
```

3) Other than the index in class, would we benefit from any other indexes on this table? Why or why not?

## Part 3: Data-driven insights

Using the table you made in part 1 and the dates you added in part 2, let's answer some questions about the bike share data

1) Build a mini-report that does a breakdown of number of trips by month
2) Build a mini-report that does a breakdown of number trips by time of day of their start and end times
3) What are the most popular stations to bike to in the summer?
4) What are the most popular stations to bike from in the winter?
5) Come up with a question that's interesting to you about this data that hasn't been asked and answer it.


