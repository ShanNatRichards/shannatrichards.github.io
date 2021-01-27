---
layout: default
title:  "Dynamic SQL with sp_executesql"
date:   2021-01-25 10:21:38 -0800
categories: tsql, programmatic sql
excerpt: "Dynamic SQL adds flexibility, modularity, and reusability to SQL coding by building SQL statements as strings at runtime and then executing. Let's explore."
---

Dynamic SQL adds flexibility, modularity, and reusability to your SQL code by building SQL statements as strings at runtime and then executing that string.
Typically, SQL statements are hard-coded. Think for example, when we create a procedure or trigger, these are compiled, stored as database objects and then when
the object is invoked, they are executed at runtime. However, with dynamic SQL we can both create and execute SQL statements at runtime. In TSQL, 
the command sp_executesql is a common way of accomplishing dynamic SQL statements. Let’s explore below.


### Scenario:
Using our cleaned and anonymized voter [datasets](https://github.com/ShanNatRichards/T-SQL/tree/main/datasets), we want to figure out the most common occupations among the voters for our district tables. 

*TLDR; See full code [here](https://github.com/ShanNatRichards/T-SQL/blob/main/dynamic_sql_example.sql).*


A first instinct maybe to run something similar to the query below – which returns the top 1 occupation for each table, performs a union, and then returns the result.



```sql

SELECT jobtitle, num_of_Voters, ‘wbw’ as [district]
FROM  (
SELECT TOP(1) Jobtitle, COUNT(*) AS [num_of_voters]
FROM wbw_voters
GROUP By Jobtitle
ORDER BY [num_of_voters] DESC, 
         Jobtitle asc          --tiebreaker
) wbw

UNION 

SELECT jobtitle, num_of_Voters, ‘gtc’ as [district]
FROM  (
SELECT TOP(1) Jobtitle,  COUNT(*) AS [num_of_voters]
FROM gtc_voters
GROUP By Jobtitle
ORDER BY [num_of_voters] DESC,
         Jobtitle asc           --tie breaker
) gtc

```
The problem with the above is that code extensibility will become difficult. Think of this: what if we added 10? 20? more district tables to our dataset? 

Then, unfortunately, we have to go in and add **lots and lots** of lines to the query in order to grab the top occupation from the new tables. The code 
would quickly become unwieldy and probably hard to maintain.

**Solution?** 
Use dynamic SQL.

Dynamic SQL is well suited for this task as it provides for code re-usability. In our scenario, code re-usability is exactly what we need as we are 
essentially repeating a procedure (finding the top occupation) but for different tables. 


### Let’s implement

1. First, convert our query as nvarchar string and account for the variations in the query. The point which varies in the query is the table name. 
Therefore, we will build our string by passing it a variable which contains a table name. 

```sql
DECLARE @sql_stmt nvarchar(300), @tablename nvarchar(50);

SET @sql_stmt =  N’SELECT TOP(1) jobtitle, COUNT(*) AS [num_of_voters] FROM  ’ + @tablename + 
                 N‘  GROUP By jobtitle  ORDER BY [num_of_voters] DESC, jobtitle asc;’ ;
```

2. Next, what we want to be returned in the query results?  
Well…we want the top 1 occupation and the num of voters with that job title. 
With dynamic SQL, we can return those results to declared variables using *string parameters*. 
Below we set up the parameters within the string itself and we also declare variables in the declaration header 

```sql
-- our results will be returned to these variables
DECLARE @sql_stmt nvarchar(500),  @tablename nvarchar(50);
DECLARE @occ varchar(50), @num_voters INT;  

--below we insert our string parameters
SET @sql_stmt =  N’SELECT TOP(1) @occ_out = jobtitle , @num_voters_out= COUNT(*) FROM ’ + @tablename + 
                 N‘ GROUP By jobtitle ORDER BY Count(*) DESC, jobtitle asc;’ ;

```

3. Now, we can set up the execution of this statement using sp_executesql. 

```sql
EXECUTE sp_executesql @stmt = @sql_stmt , 
                   @params = N'@occ_out varchar(50) OUTPUT, @num_voters_out int OUTPUT', 
                   @occ_out = @occ OUTPUT, 
                   @num_voters_out = @num_voters OUTPUT;
```
Let’s explain a bit of the above. 
sp_executesql takes multiple arguments. The first argument @stmt, takes the sql statement string we have built. 
The argument @params tells the sp_executesql command that our string uses parameters. It also defines the datatypes of those parameters and signals whether they are used to return (OUTPUT) values. 
For example, for our @occ_out parameter, we define its datatype (based off what will be returned from the table) and signal that it will be an output parameter. 
The subsequent arguments indicate the variables to which each of our output parameters will return results. For example, the parameter @num_voters_out will send output to the variable @num_voters. 


4. Now we have the dynamic string and execution commands set, we can make our script procedural by putting our code in a procedure (or a function would work as well).

```sql

CREATE PROC  getTopOccupation ( @tablename nvarchar(50) )
AS
BEGIN
DECLARE @sql_stmt nvarchar(500),  @tablename nvarchar(50);
DECLARE @occ varchar(50), @num_voters INT;  

SET @sql_stmt =  N’SELECT TOP(1) @occ_out = jobtitle , @num_voters_out= COUNT(*) FROM ’ + @tablename + 
                 N‘ GROUP By jobtitle ORDER BY Count(*) DESC, jobtitle asc;’ ;
EXECUTE sp_executesql @stmt = @sql_stmt , 
                   @params = N'@occ_out varchar(50) OUTPUT, @num_voters_out int OUTPUT', 
                  @occ_out = @occ OUTPUT, 
                   @num_voters_out = @num_voters OUTPUT;

SELECT @occ, @num_voters;
END;

```

Now, we have a re-usable stored procedure that can scale with the number of voter districts we get datasets for. 
We simply pass in the tablename and have the stored procedure programmatically return the result.

```sql
EXEC getTopOccupation  @tablename = N‘wcw_voters’;
```

### Conclusion
Dynamic SQL with sp_executesql adds reusability and extensibility to our SQL coding. It can be immensely  useful  for, example,  when we are repeating a similar task for different database objects. 







