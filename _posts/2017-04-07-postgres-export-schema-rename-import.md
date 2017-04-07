---
layout: post
title: "Postgresql. Export schema, rename and import"
date: '2017-04-07'
author: Juri Timošin
tags:
- postgresql
---

Lately as a process of joining different Postgres databases with 1 schema each into 1 database with
different schemas, I had to export a schema from a database and to import it into another database
with a different schema name.

<!--more-->

Turned out postgres does not support setting schema name on import or export, so I had to export to
plain text file, edit it and then import. But actually we can do it on the fly too by using a
_simple_ onliner. The key here are the options:

- we assign PGPASSWORD environment variable, so that psql does not ask for a password.
- we dump only 1 schema (`-n public`)
- we do not store ownership (`-O`), privileges (`-x`), table spaces and security labels, because
those will be different in the new database.
- we pipe the plain text dump to `sed` and find the line where postgres sets current schema and
replace it with creating a new schema and setting it as current for the import.
- and in the end we import the dump again using the PGPASSWORD environment variable, also setting
ON_ERROR_STOP to 1, so that the dump stops, if some error occures.


{% highlight shell %}
PGPASSWORD=<password> pg_dump -h <host> -U <user> -d <db> --no-tablespaces --no-security-labels -xOn public |
sed 's/SET search_path = public, pg_catalog;/DROP SCHEMA <new_schema> CASCADE; CREATE SCHEMA <new_schema>; SET search_path = <new_schema>;/' |
PGPASSWORD=<password> psql -h <host> -U <user> -d <db> -v ON_ERROR_STOP=1
{% endhighlight %}
