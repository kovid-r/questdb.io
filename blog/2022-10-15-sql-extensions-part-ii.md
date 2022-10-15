---
title: SQL extensions for time series data in QuestDB part II
author: Kovid Rathee
author_title: Guest post
author_url: https://kovidrathee.medium.com/
author_image_url: https://miro.medium.com/fit/c/96/96/0*_CwYR2OmNap47tQO.jpg
description:
  SQL extensions for time series data in QuestDB part II:
  - tutorial
  - sql
  - questdb
  - timeseries

image: /img/blog/2022-08-05/banner.png
tags: [tutorial, sql, timeseries, questdb]
---

This post comes from [Kovid Rathee](https://towardsdatascience.com/questdb-vs-timescaledb-38160a361c0e?sk=42d1c037a6dfc3786e11eb9d9f5af2ad), who, following up on the [first tutorial](https://towardsdatascience.com/sql-extensions-for-time-series-data-in-questdb-f6b53acf3213), has put together another tutorial on SQL extensions for time series data in QuestDB.

<!--truncate-->

# Draft / SQL extensions for time series data in QuestDB part II

This tutorial follows up on the one where we introduced [SQL extensions in QuestDB](https://towardsdatascience.com/sql-extensions-for-time-series-data-in-questdb-f6b53acf3213) that make time-series analysis easier. In this tutorial, you will learn in detail about the [`SAMPLE BY` extension](https://questdb.io/docs/reference/sql/sample-by/) in QuestDB, which will enable you to work with time-series data efficiently because of its simplicity and flexibility.

To get started with this tutorial, you should know that `SAMPLE BY` is a SQL extension in QuestDB that helps you group or bucket your time-series data based on the [designated timestamp](https://questdb.io/docs/concept/designated-timestamp). It removes the need for lengthy `CASE WHEN` statements and `GROUP BY` clauses. Not only that, the `SAMPLE BY` extension helps you quickly deal with many other data-related issues, such as [missing data](https://questdb.io/docs/reference/sql/select/#fill), [incorrect timezones](https://questdb.io/docs/reference/sql/sample-by/#align-to-calendar-time-zone), and [offsets](https://questdb.io/docs/reference/sql/sample-by/#align-to-calendar-with-offset).

This tutorial assumes you have an up-and-running QuestDB instance ready for use. Let's dive straight into it.

## Setup

### Import Sample Data

Similar to the previous tutorial, we'll use [the NYC taxi riders data for February 2018](https://s3-eu-west-1.amazonaws.com/questdb.io/datasets/grafana_tutorial_dataset.tar.gz). You can use the following script utilizing the [HTTP REST API](https://questdb.io/docs/guides/importing-data-rest/) to upload data into QuestDB:

```sh
curl https://s3-eu-west-1.amazonaws.com/questdb.io/datasets/grafana_tutorial_dataset.tar.gz > grafana_data.tar.gz
tar -xvf grafana_data.tar.gz

curl -F data=@taxi_trips_feb_2018.csv http://localhost:9000/imp
curl -F data=@weather.csv http://localhost:9000/imp  
```
 
Alternatively, you can utilize [the import functionality in the QuestDB console](https://questdb.io/docs/develop/web-console/#import), as shown in the image below:

![](https://i.imgur.com/EWniDQq.png)

For [importing large CSV files into partitioned tables](https://questdb.io/docs/guides/importing-data/), QuestDB recommends using the `COPY` command. Thie method is especially useful when you are trying to migrate data from another database into QuestDB.

### Create an Ordered Timestamp Column

QuestDB mandates the use of an ordered timestamp column, so you'll have to cast the `pickup_datetime` column to `TIMESTAMP` in a new table called `taxi_trips` with the script below:

```sql
CREATE TABLE taxi_trips AS (
  SELECT * 
    FROM 'taxi_trips_feb_2018.csv' 
   ORDER BY pickup_datetime
) TIMESTAMP(pickup_datetime) 
PARTITION BY MONTH;
```

> By converting the `pickup_datetime` column to timestamp, you'll allow QuestDB to use it as the [designated timestamp](https://questdb.io/docs/concept/designated-timestamp/). Using the designated timestamp column, QuestDB is able to index the table to run time-based queries more efficiently.

If it all goes well, you should see the following data after running a `SELECT *` query on the `taxi_trips` table:

![](https://i.imgur.com/QwI0YVe.png)

## Understanding the Basics of `SAMPLE BY`

The `SAMPLE BY` extension allows you to create groups and buckets of data based on time ranges. This is especially valuable for time-series data as you can calculate frequently used aggregates with extreme simplicity. `SAMPLE BY` offers you the ability to summarize or aggregate data from very fine to very coarse [units of time](https://questdb.io/docs/reference/sql/sample-by/#sample-units), i.e., from microseconds to months and everything in between, i.e., millisecond, second, minute, hour, and day. You can derive other units of time, such as a week, fortnight, and year from the ones provided out of the box.

Let's look at some examples to understand how to use `SAMPLE BY` in different scenarios.

### Hourly Count of Trips

You can use the `SAMPLE BY` keyword with the [sample unit](https://questdb.io/docs/reference/sql/sample-by/#sample-units) of `h` to get an hour-by-hour count of trips for the whole duration of the data set. Running the following query, you'll get results in the console:

```sql
SELECT pickup_datetime,
       COUNT() total_trips
  FROM 'taxi_trips'
 SAMPLE BY 1h;
```

There are two ways you can read your data in the QuestDB console: using the grid, which has a tabular form factor or using a chart, where you can draw up a line chart, a bar graph, or an area chart to [visualize your data](https://questdb.io/docs/develop/web-console/#visualizing-results). Here's an example of a bar chart drawn from the query mentioned above:

![](https://i.imgur.com/JHBiCI3.png)

### Three-Hourly Holistic Summary of Trips

The `SAMPLE BY` extension allows you to group data by any arbitrary number of sample units. In the following example, you'll see that the query is calculating a three-hourly summary of trips with multiple aggregate functions:

```sql
SELECT pickup_datetime,
       COUNT() total_trips,
       SUM(passenger_count) total_passengers,
       ROUND(AVG(trip_distance), 2) avg_trip_distance,
       ROUND(SUM(fare_amount)) total_fare_amount,
       ROUND(SUM(tip_amount)) total_tip_amount,
       ROUND(SUM(fare_amount + tip_amount)) total_earnings
  FROM 'taxi_trips'
 SAMPLE BY 3h;
 ```
You can view the output of the query in the following grid on the QuestDB console:
 
 ![](https://i.imgur.com/NG2sDIV.png)

### Weekly Summary of Trips

As mentioned earlier in the tutorial, although there's no sample unit for a week, a fortnight, or a year, you can derive them simply by utilizing the built-in sample units. If you want to sample the data by a week, use `7d` as the sampling time, as shown in the query below:

```sql
SELECT pickup_datetime,
       COUNT() total_trips,
       SUM(passenger_count) total_passengers,
       ROUND(AVG(trip_distance), 2) avg_trip_distance,
       ROUND(SUM(fare_amount)) total_fare_amount,
       ROUND(SUM(tip_amount)) total_tip_amount,
       ROUND(SUM(fare_amount + tip_amount)) total_earnings
  FROM 'taxi_trips'
 WHERE pickup_datetime BETWEEN '2018-02-01' AND '2018-02-28'
 SAMPLE BY 7d;
```

![](https://i.imgur.com/f5lVlQL.png)

## Dealing with Missing Data

If you've worked a fair bit with data, you already know that data isn't always in a pristine state. One of the most common issues, especially with time-series data, is discontinuity, i.e., scenarios where data is missing for specific time periods. You can quickly identify and deal with missing data using the advanced functionality of the `SAMPLE BY` extension.

QuestDB offers an easy way to generate and fill missing data with the `SAMPLE BY` clause. Take the following example: I've deliberately removed data from 4 am to 5 am for the 1st of February 2018. Notice how the [`FILL` keyword](https://questdb.io/docs/reference/sql/fill/), when used in conjunction with the `SAMPLE BY` extension, can generate a row for the hour starting at 4 am and fill it with some data:

```sql
SELECT pickup_datetime,
       COUNT() total_trips,
       SUM(passenger_count) total_passengers,
       ROUND(AVG(trip_distance), 2) avg_trip_distance,
       ROUND(SUM(fare_amount)) total_fare_amount,
       ROUND(SUM(tip_amount)) total_tip_amount,
       ROUND(SUM(fare_amount + tip_amount)) total_earnings
  FROM 'taxi_trips'
 WHERE pickup_datetime NOT BETWEEN '2018-02-01T04:00:00' AND '2018-02-01T04:59:59'
 SAMPLE BY 1h FILL(LINEAR);
```
![](https://i.imgur.com/8hD7Lmw.png)

In the example above, we've used an inline `WHERE` clause to emulate missing clause with the help of the `NOT BETWEEN` keyword. Alternatively, you can create a separate table with missing trips using the same idea, as shown below:

```sql 
CREATE TABLE 'taxi_trips_missing' AS (
SELECT * FROM 'taxi_trips'
WHERE pickup_datetime NOT BETWEEN '2018-02-01T04:00:00' 
  AND '2018-02-01T04:59:59');
```

The [`FILL`](https://questdb.io/docs/reference/sql/sample-by/#fill-options) keyword demands a `fillOption` from the following:

| `fillOption` | Usage scenario | Notes |
| -------- | -------- | -------- |
| NONE     | When you don't want to populate missing data, and leave it as is | This is the default `fillOption` |
| NULL     | When you want to generate rows for missing time periods, but leave all the values as NULLs | |
| PREV     | When you want to copy the values of the previous row from the summarized data | This is useful when you expect the numbers to be similar to the preceding time period |
| LINEAR   | When you want to normalize the missing values, you can take the average of the immediately preceding and following row | |
| CONST or x | When you want to hardcode values where data is missing  | FILL (column_1, column_2, column_3, ...)     |

Here's another example of hardcoding values using the FILL(x) `fillOption`:

![](https://i.imgur.com/gN0LO6g.png)

## Working with Timezones and Offsets

The `SAMPLE BY` extension also enables you to work change timezones and add or subtract offsets from your timestamp columns to adjust for any issues you might encounter when dealing with different source systems, especially in other geographic areas. It is important to note that, by default, QuestDB aligns its [sample calculation](https://questdb.io/docs/reference/sql/sample-by/#sample-calculation) based on the `FIRST OBSERVATION`, as shown in the example below:

```sql
SELECT pickup_datetime,
       COUNT() total_trips,
       SUM(passenger_count) total_passengers,
       ROUND(AVG(trip_distance), 2) avg_trip_distance,
       ROUND(SUM(fare_amount)) total_fare_amount,
       ROUND(SUM(tip_amount)) total_tip_amount,
       ROUND(SUM(fare_amount + tip_amount)) total_earnings
  FROM 'taxi_trips'
 WHERE pickup_datetime BETWEEN '2018-02-01T13:35:52' AND '2018-02-28'
 SAMPLE BY 1d;
```

![](https://i.imgur.com/U9m6k6s.png)

Note now the `1d` sample calculation starts at `13:35:52` and ends at `13:35:51` the next day. Apart from the one demonstrated above, there are two other ways to align your sample calculations -- to the [`calendar time zone`](https://questdb.io/docs/reference/sql/sample-by/#align-to-calendar-time-zone), and to [`calendar with offset`](https://questdb.io/docs/reference/sql/sample-by/#align-to-calendar-with-offset).

Let's look at the other two alignment methods now.

### Aligning Sample Calculation to Another Timezone

When moving data from one system to another or via a complex pipeline, you can encounter issues with time zones. For the sake of demonstration, let's assume that you've identified that the data set you've loaded into the database is not for New York City but for Melbourne, Australia. These two cities are far apart and are in very different time zones.

QuestDB allows you to fix this issue by aligning your data to another timezone using the [`ALIGN TO CALENDAR TIME ZONE` option](https://questdb.io/docs/reference/sql/sample-by/#align-to-calendar-time-zone) with the `SAMPLE BY` extension. In the example shown below, you can see how an `ALIGN TO CALENDAR TIME ZONE ('AEST')` has helped align the `pickup_datetime`, i.e., the designated timestamp column to the AEST timezone for Melbourne.

```sql
SELECT pickup_datetime,
       COUNT() total_trips,
       SUM(passenger_count) total_passengers,
       ROUND(AVG(trip_distance), 2) avg_trip_distance,
       ROUND(SUM(fare_amount)) total_fare_amount,
       ROUND(SUM(tip_amount)) total_tip_amount,
       ROUND(SUM(fare_amount + tip_amount)) total_earnings
  FROM 'taxi_trips'
 SAMPLE BY 3h
 ALIGN TO CALENDAR TIME ZONE ('AEST');

```

![](https://i.imgur.com/szB7CMD.png)

### Aligning Sample Calculation with Offsets

Similar to the previous example, you can also align your sample calculation by [offsetting the designated timestamp](https://questdb.io/docs/reference/sql/sample-by/#align-to-calendar-with-offset) column manually by any `hh:mm` value between -23:59 to 23:59. In the following example, we're offsetting the sample calculation by -5:30, i.e., negative five hours and thirty minutes:

```sql
SELECT pickup_datetime,
       COUNT() total_trips,
       SUM(passenger_count) total_passengers,
       ROUND(AVG(trip_distance), 2) avg_trip_distance,
       ROUND(SUM(fare_amount)) total_fare_amount,
       ROUND(SUM(tip_amount)) total_tip_amount,
       ROUND(SUM(fare_amount + tip_amount)) total_earnings
  FROM 'taxi_trips'
 SAMPLE BY 3h
 ALIGN TO CALENDAR WITH OFFSET '-05:30';
```

![](https://i.imgur.com/3xyC6kt.png)

## Conclusion

In this tutorial, you learned how to exploit the [`SAMPLE BY` extension](https://questdb.io/docs/reference/sql/sample-by) in QuestDB to work efficiently with time-series data, especially in aggregated form. In addition, the `SAMPLE BY` extension also allows you to fix specific common problems with time-series data attributable to complex data pipelines, disparate source systems in different geographical areas, software bugs, etc. All in all, SQL extensions like `SAMPLE BY` provide a significant advantage when working with time-series data by enabling you to achieve more in fewer lines of SQL.
