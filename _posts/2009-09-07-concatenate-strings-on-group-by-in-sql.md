---
layout: post
title: Concatenate Strings On Group By In SQL
author: Juri Timošin
tags:
- sql
- postgresql
- mysql
- mssql
---

[1]: http://dev.mysql.com/doc/refman/5.0/en/group-by-functions.html#function_group-concat
[2]: http://www.postgresql.org/docs/8.3/static/functions-string.html

Imagine we have the following table.
{% highlight sql %}
CREATE TABLE MyTable (id int, name varchar, value int);

INSERT INTO MyTable (id,name,value) VALUES (1, 'Hello', 4);
INSERT INTO MyTable (id,name,value) VALUES (1, 'World', 8);
INSERT INTO MyTable (id,name,value) VALUES (5, 'Great!', 9);
{% endhighlight %}

The result we would like to acquire is:
{% highlight console %}
| id |   name_values    |
+----+------------------+
|  1 | Hello:4; World:8 |
|  5 | Great!:9         |
{% endhighlight %}

Names and values are concatenated into strings and grouped by id. We need an aggregate function,
that concatenates strings for that. Here are some solutions for different sql databases.

<!--more-->

## MySql

This case is most easy one. Lucky users already have the [GROUP_CONCAT(expr)][1] function. This
query should give the answer.

{% highlight sql %}
SELECT id, GROUP_CONCAT(name + ':' + value SEPARATOR '; ') AS name_values
FROM MyTable GROUP BY id;
{% endhighlight %}

## PostgreSql

The solution here is a bit more difficult, but nevertheless easy enough. We need to create our own
aggregate function and use it in our query.

{% highlight sql %}
CREATE AGGREGATE aggr_textcat(
  basetype    = text,
  sfunc       = textcat,
  stype       = text,
  initcond    = ''
);

SELECT id, substring(aggr_textcat(', ' || name || ':' || value) from 2)
AS name_values FROM MyTable GROUP BY id;
{% endhighlight %}

Here we used already existing function to concatenate text fields [textcat][2], but we could write
our own.

## MsSql

Since version 2005 it became also possible to write your own aggregate function in MsSql, but here
I provide another solution using inner select and xml path.

{% highlight sql %}
SELECT id, SUBSTRING((SELECT '; ' + name + ':' + CAST(value AS varchar(MAX))
FROM MyTable WHERE (id = Results.id) FOR XML PATH ('')),3,9999) AS name_values
FROM MyTable Results
GROUP BY id;
{% endhighlight %}
