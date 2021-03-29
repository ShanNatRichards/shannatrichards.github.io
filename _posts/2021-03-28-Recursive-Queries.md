---
layout: default
title:  "Recursive Queries"
date:   2021-03-27 17:21:38 -0800
categories: postgres, sql, recursion, cte
excerpt: "In postgreSQL, we can accomplish recursive queries with the use of CTEs (common table expressions) and the UNION ALL clause. Let’s explore."
---
A recursive function is one that calls itself –  it requires a base case and a recursive call. Typically, recursive functions are useful for solving Math factorials and traversing structures such as trees and graphs, to name a few practical applications.  In postgreSQL, we accomplish recursive queries with the use of CTEs (common table expressions) and the UNION ALL clause. Let’s explore more below.

**TL;DR? See complete code [here.](https://github.com/ShanNatRichards/postgreSQL/recursive_query.sql)**

### Basics of A Recursive SQL Query

Let’s start with an elementary example of a recursive query in postgres – one which returns even numbers from 0 to 20.  

```sql
WITH RECURSIVE rcte as (
SELECT 0 AS even_num  

UNION 

SELECT even_num+2
FROM rcte 
WHERE even_num < 20
)

SELECT even_num from rcte;
```
Within our cte, the first SELECT is the non-recursive term - the simplest form of the query that does not need a recursive call.  In this case, it sets the first even number to 0.   

In the second SELECT, our recursive calls take place. Notice that in this SELECT query, the FROM clause references the cte itself *rcte*. In essence, the query is calling itself. Furthermore, everytime it does a recursive call, it adds 2 to the previous number.

Let’s see our results below. 

![Recursion Even Numbers](https://raw.githubusercontent.com/ShanNatRichards/postgreSQL/main/images/even_numbers.png)

 
Our recursive query returns rows with even numbers from 0 to 20.

### Recursion in Practice
Now that we’ve demonstrated a basic case of a SQL recursion – how would a recursive CTE be useful in practice? 
As mentioned briefly, recursive algorithms are great for traversing trees and graphs. A [popular example](https://www.dbta.com/Columns/DBA-Corner/An-Introduction-to-Recursive-SQL-96878.aspx) of a recursive cte in practice is that of recursing through a table that stores an organization’s employees with bosses, execs, middle managers, ...etc. to figure out organizational hierarchy.
For our example, we’re going to traverse through possible activity sequences (likes nodes in a graph) to come up with potential tour schedules.

***The Context*** 

Let’s consider a start-up tour company, Excellent Tours, that plans to offer a fun-filled city tour. 
Currently, the company is in the process of figuring out tour activities/stops and, ultimately, the tour schedule. The clerks for the company have been researching city activities and then storing that those potential tour stops in a database table called tour_stops.
 

```sql
CREATE TABLE tour_stops (
stop_id INT,
name VARCHAR(100),
start_time TIME,
end_time TIME
);
```

***The Problem***

The company’s clerks, in a bid to ensure tour variety, have identified and stored way too many potential tour-stops. Now, the company manager has to figure out the best sequence of activities to include in the tour.  For example, if the manager includes  ‘Painting at the Museum’ from 2pm – 4pm, then the tour cannot include activities that overlap in that timeframe such as ‘Visiting Zoo’ from 1pm – 3pm.  
So, we need to figure out possible activity plans for a tour, where the activity time frames do not overlap. Also, the manager is keen start the tour with Yoga in the park. 

Here’s our table:

![table](https://github.com/ShanNatRichards/postgreSQL/main/images/tour_stop%20table.png)


***The Solution***

Let’s use a recursive query to spool out possible activity sequences for the tour. 

```sql
WITH RECURSIVE stg as (
SELECT start_time
, end_time
, concat('>> ' , name, ' ', start_time, ' - ', end_time) as activity_plan
FROM tour_stops
WHERE stop_id = 14

UNION ALL

SELECT t.start_time
, t.end_time
, concat(stg.activity_plan, '>> ', t.name, ' ', t.start_time, ' - ', t.end_time) as activity_plan 
FROM stg
INNER JOIN tour_stops t ON t.start_time > stg.end_time
)

SELECT activity_plan
FROM stg;
```
The recursion returns 286 rows and here’s a sample of the results:
![recursive query](https://github.com/ShanNatRichards/postgreSQL/blob/main/images/result1.png)
 
Let’s go over what’s happening above.

In the initial SELECT query, we return one row for Yoga in the Park – which is the activity the manager is keen to start the tour with. This is row 1 in the results above. 
In the recursive SELECT, there is a JOIN back to the tour_stop table, which looks for rows that have a start time greater than the end time for ‘Yoga in the Park’. There are several activities that meet this condition and are returned in rows 2 -18.
For every row returned from 2- 18, the recursion will be called again looking for activities with start times greater than the end times of those activities in 2-18. And so on and so forth until the recursion can longer find rows that meet the join condition. 


***Refining The Query***

The query returned quite a long list of potential schedules for the tour. However, most of these returned rows are superfluous and wouldn’t be useful to the tour manager. We can refine the returned list to be more useful by adding  extra filters. 
Let’s say the manager wants activity sequences that includes lunch. Also, the final activity in the sequence should end after 3pm.

In the outer SELECT clause, let’s add a WHERE clause and also re-order the query output so that we see the longest chain of activities first.

```sql
SELECT activity_plan
FROM stg
WHERE  end_time > '15:00' 
       AND activity_plan like '%Lunch%'
ORDER BY LENGTH(activity_plan) DESC;
```

Sample of Results:
 
![recursive query 2](https://github.com/ShanNatRichards/postgreSQL/blob/main/images/result_cleaned.png)

### Concluding Considerations:

Recursion queries can be very expensive – especially when the stopping conditions aren’t clearly considered. Imagine if this table actually had thousands of records? Or worse, a clerk made an error and entered an activity which had end time that was less than the start time. Your recursion could easily spool out of control. So, it’s important to use recursions in a careful and considered manner. 


  

