---
layout: page
title: Databases and SQL
subtitle: Creating and Modifying Data
minutes: 30
---
> ## Learning Objectives {.objectives}
>
> *   Write statements that creates tables.
> *   Write statements to insert, modify, and delete records.

So far we have only looked at how to get information out of a database,
both because that is more frequent than adding information,
and because most other operations only make sense
once queries are understood.
If we want to create and modify data,
we need to know two other sets of commands.

The first pair are [`CREATE TABLE`][CREATE-TABLE] and [`DROP TABLE`][DROP-TABLE].
While they are written as two words,
they are actually single commands.
The first one creates a new table;
its arguments are the names and types of the table's columns.
For example,
the following statements create the four tables in our survey database:

~~~ {.sql}
CREATE TABLE Person(ident TEXT, personal TEXT, family TEXT);
CREATE TABLE Site(name TEXT, lat REAL, long REAL);
CREATE TABLE Visited(ident INTEGER, site TEXT, dated TEXT);
CREATE TABLE Survey(taken INTEGER, person TEXT, quant REAL, reading REAL);
~~~

We can get rid of one of our tables using:

~~~ {.sql}
DROP TABLE Survey;
~~~

Be very careful when doing this:
most databases have some support for undoing changes,
but it's better not to have to rely on it.

Different database systems support different data types for table columns,
but most provide the following:

data type  use
---------  -----------------------------------------
INTEGER    a signed integer
REAL       a floating point number
TEXT       a character string
BLOB       a "binary large object", such as an image

Most databases also support Booleans and date/time values;
SQLite uses the integers 0 and 1 for the former,
and represents the latter as discussed [earlier](#a:dates).
An increasing number of databases also support geographic data types,
such as latitude and longitude.
Keeping track of what particular systems do or do not offer,
and what names they give different data types,
is an unending portability headache.

When we create a table,
we can specify several kinds of constraints on its columns.
For example,
a better definition for the `Survey` table would be:

~~~ {.sql}
CREATE TABLE Survey(
    taken   integer not null, -- where reading taken
    person  text,             -- may not know who took it
    quant   real not null,    -- the quantity measured
    reading real not null,    -- the actual reading
    primary key(taken, quant),
    foreign key(taken) references Visited(ident),
    foreign key(person) references Person(ident)
);
~~~

Once again,
exactly what constraints are available
and what they're called
depends on which database manager we are using.

Once tables have been created,
we can add, change, and remove records using our other set of commands,
`INSERT`, `UPDATE`, and `DELETE`.

The simplest form of `INSERT` statement lists values in order:

~~~ {.sql}
INSERT INTO Site values('DR-1', -49.85, -128.57);
INSERT INTO Site values('DR-3', -47.15, -126.72);
INSERT INTO Site values('MSK-4', -48.87, -123.40);
~~~

We can also insert values into one table directly from another:

~~~ {.sql}
CREATE TABLE JustLatLong(lat text, long text);
INSERT INTO JustLatLong SELECT lat, long FROM site;
~~~

Modifying existing records is done using the `UPDATE` statement.
To do this we tell the database which table we want to update,
what we want to change the values to for any or all of the fields,
and under what conditions we should update the values.

For example, if we made a mistake when entering the lat and long values
of the last `INSERT` statement above:

~~~ {.sql}
UPDATE Site SET lat=-47.87, long=-122.40 WHERE name='MSK-4'
~~~

Be careful to not forget the `where` clause or the update statement will
modify *all* of the records in the database.

Deleting records can be a bit trickier,
because we have to ensure that the database remains internally consistent.
If all we care about is a single table,
we can use the `DELETE` command with a `WHERE` clause
that matches the records we want to discard.
For example,
once we realize that Frank Danforth didn't take any measurements,
we can remove him from the `Person` table like this:

~~~ {.sql}
DELETE FROM Person WHERE ident = 'danforth';
~~~

But what if we removed Anderson Lake instead?
Our `Survey` table would still contain seven records
of measurements he'd taken,
but that's never supposed to happen:
`Survey.person` is a foreign key into the `Person` table,
and all our queries assume there will be a row in the latter
matching every value in the former.

This problem is called [referential integrity](reference.html#referential-integrity):
we need to ensure that all references between tables can always be resolved correctly.
One way to do this is to delete all the records
that use `'lake'` as a foreign key
before deleting the record that uses it as a primary key.
If our database manager supports it,
we can automate this
using [cascading delete](reference.html#cascading-delete).
However,
this technique is outside the scope of this chapter.

> ## Hybrid Storage Models {.callout}
>
> Many applications use a hybrid storage model
> instead of putting everything into a database:
> the actual data (such as astronomical images) is stored in files,
> while the database stores the files' names,
> their modification dates,
> the region of the sky they cover,
> their spectral characteristics,
> and so on.
> This is also how most music player software is built:
> the database inside the application keeps track of the MP3 files,
> but the files themselves live on disk.

> ## Replacing NULL {.challenge}
>
> Write an SQL statement to replace all uses of `null` in
> `Survey.person` with the string `'unknown'`.

> ## Generating Insert Statements {.challenge}
>
> One of our colleagues has sent us a [CSV](reference.html#comma-separated-values) file containing
> temperature readings by Robert Olmstead, which is formatted like
> this:
>
> ~~~ {.output}
> Taken,Temp
> 619,-21.5
> 622,-15.5
> ~~~
>
> Write a small Python program that reads this file in and prints out
> the SQL `INSERT` statements needed to add these records to the
> survey database.  Note: you will need to add an entry for Olmstead
> to the `Person` table.  If you are testing your program repeatedly,
> you may want to investigate SQL's `INSERT or REPLACE` command.

> ## Backing Up with SQL {.challenge}
>
> SQLite has several administrative commands that aren't part of the
> SQL standard.  One of them is `.dump`, which prints the SQL commands
> needed to re-create the database.  Another is `.load`, which reads a
> file created by `.dump` and restores the database.  A colleague of
> yours thinks that storing dump files (which are text) in version
> control is a good way to track and manage changes to the database.
> What are the pros and cons of this approach?  (Hint: records aren't
> stored in any particular order.)

[CREATE-TABLE]: https://www.sqlite.org/lang_createtable.html
[DROP-TABLE]: https://www.sqlite.org/lang_droptable.html
