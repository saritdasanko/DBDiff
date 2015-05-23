# DBDiff CLI - Database Diff Command Line Interface

#The Problem

I've found a few solid​ ​migrat​ion tools like Flyway (https://github.com/flyway/flyway) and Simple DB Migrate (https://github.com/guilhermechapiewski/simple-db-migrate)​, the latter being my preference for it's simplicity​ but the former having a lot more commits and contributors.​ However,​ these are just migrators and do not help produce the actual diff/migration to be versioned​, which I would not like to do manually if it can be automated.​

I found a diff tool by MySQL called mysqldbcompare (http://dev.mysql.com/doc/mysql-utilities/1.6/en/mysqldbcompare.html), which outputs SQL​ for (some) schema and data changes​ but it​ shockingly doesn't​ ​produce valid SQL!!!

I must say though everything the mysqldbcompare tool offers is great, doing the diff by connecting directly to a source and target database (can be locally or on another server) and doing a series of tests/checks (all of which can be skipped if need be), then checking both the schema and the data - but we just need it to produce valid SQL output and I think it also had trouble in validating the schema fully.

​​I actually looked through A LOT of ​different ​schema and data diff tools (took me a whole weekend) and they all either:

1. Only do schema diffs
2. Are just too complex to get working or
3. Just don't produce valid SQL

#The Solution
I think a solid migration tool mixed with an automated schema and data diff tool would be a great contribution to the open source community.

This is what DBDiff is.

#Features of DBDiff
-   Works on Windows, Linux & Mac command-line/Terminal because it has been developed in PHP
-   Connects to a source and target database to do the comparison diff, locally and remotely
-   Diffs can include changes to the schema and/or data, both in valid SQL to bring the target up-to-date with the source
-   Diffs are SUPER fast and this tool has been tested with databases of multiple tables of millions of rows
-   Since this diff tool is being used for migrations, it provides up and down SQL in the same file
-   Works with existing migration tools like Flyway and Simple DB Migrate by specifying output template files/formats, for example, Simple DB Migrate may work with simple-db-migrate.tmpl which includes: "SQL\_UP = u""" {{ up }} """ SQL\_DOWN = u""" {{ down }} """
-   Is Unicode aware, can work with UTF8 data, which includes foreign characters/symbols
-   Works with just MySQL for now, but can easily be expandable to other DBs in the 

# Command-Line API

Here is the suggested API this dbdiff tool should have, which is very similar to the mysqldbcompare tool mentioned earlier. You may add or modify this if you think there are better alternatives and/or missing parameters:

-   --server1=user:password@host1:port - Specify the source db connection details. If there is only one server the --server1 flag can be omitted
-   --server2=user:password@host2:port - Specify the target db connection details (if it’s different to server1)
-   --format=sql - sql should be the default
-   --template=templates/simple-db-migrate.tmpl - Specifies the output template, if any. By default will be plain SQL
-   --type=schema or data or all - Specifies the type of diff to do either on the schema, data or both. schema is the default
-   --include=up or down or all - Specified whether to include the up, down or both data in the output. up is the default
-   --no-comments=false - By default automated comments starting with the hash (\#) character are included in the output file, which can be removed with this parameter
-   --config=config.yaml - By default, DBDiff will look for a .dbdiff file in the current directory which is valid YAML, which may also be overridden with a config file that lists the database host, user, port and password of the source and target DBs in YAML format (instead of using the command line for it), or any of the other settings e.g. the format, template, type, include, no-comments. Please note: a command-line parameter will always override any config file.
-   server1.db1.table1:server2.db2.table3 or server1.db1:server2.db2 - The penultimate parameter is what to compare. This tool can compare just one table or all tables (entire db) from the database
-   --output=./output-dir/today-up-schema.sql - The last parameter is an output file and/or directory to output the diff to, which by default will output to the same directory the command is run in if no directory is specified. If a directory is specified, it should exist, otherwise an error will be thrown. If this path is not specified, the default file name should be migration.sql in the current directory

# Usage Examples

## Example 1
\$ ./dbdiff server1.db1:server2.db2

This would by default look for the .dbdiff config file for the DB connection details, if it’s not there the tool would return an error. If it’s there, the connection details would be used to compare the SQL of only the schema and output a commented .sql file inside the current directory which includes only the up SQL as per default

## Example 2
\$ ./dbdiff server1.development.table1:server2.production.table1 --no-comments --type=data

This would by default look for the .dbdiff config file for the DB connection details, if it’s not there the tool would return an error. If it’s there, the connection details would be used to compare the SQL of only the data of the specified table1 inside each database and output a .sql file which has no comments inside the current directory which includes only the up SQL as per default

## Example 3
\$ ./dbdiff --config=config.conf --template=templates/simple-db-migrate.tmpl --include=all server1.db1:server2.db2 ./sql/simple-schema.sql

Instead of looking for .dbdiff, this would look for config.conf (which should be valid YAML) for the settings, and then override any of those settings from config.conf for the --template and --include parameters given in the command-line parameters - thus comparing the source file db1.sql with db2.sql and outputting an SQL file called simple-schema.sql to the ./sql folder, which should already exist otherwise the program will throw an error, and which includes only the schema as an up and down SQL diff in the simple-db-migrate format (as specified by the template). This example would work perfectly alongside the simple-db-migrate tool

# File Examples

## .dbdiff

		server1-user: user
		server1-password: password
		server1-port: port
		server1-host: host1
		template: templates/simple-db-migrate.tmpl
		type: all
		include: all
		no-comments: true

## simple-db-migrate.tmpl

		SQL\_UP = u"""

		{{ up }}

		"""

		SQL\_DOWN = u"""

		{{ down }}

		"""

# How Does the Diff Actually Work?

The following comparisons run in exactly the following order:

-   When comparing multiple tables: all comparisons should be run
-   When comparing just one table with another: only run the schema and data comparisons

## Overall Comparison
-   Check both databases exist and are accessible, if not, throw an error
-   The database collation is then compared between the source and the target and any differences noted for the output

## Schema Comparison
-   Looks to see if there are any differences in column numbers, name, type, collation or attributes
-   Any new columns in the source, which are not found in the target, are added

## Data Comparison
-   And then for each table, the table storage type (e.g. MyISAM, CSV), the collation (e.g. utf8\_general\_ci), and number of rows are compared, in that order. If there are any differences they are noted before moving onto the next test
-   Next, both changed rows as well as missing rows from each table are recorded
