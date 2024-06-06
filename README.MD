# GUIDE ON DATABASE MIGRATION ON EXISTING DATABASE WITH LIQUIBASE

1. Start db

2. Check db up

```psql -U postgres -h localhost test_db```

3. Given data already exists (simulate like so):

``` 
set schema 'test_schema';
CREATE TABLE first_table (
   id bigint primary key,
   name varchar
);
```

4. Make sure liquibase.properties and postgres driver jar are in
   your project folder and liquibase.properties points to them

5. Create initial changelog with the current state of the database (init_changelog.xml), run generateChangeLog, also referred to as creating a "snapshot" of the db. Include all possible diff types including table data:

```liquibase --defaultSchemaName=test_schema --changeLogFile=init_changelog.xml --diff-types="data,catalogs,checkconstraints,columns,databasepackage,databasepackagebody,foreignkeys,functions,indexes,primarykeys,sequences,storedprocedures,tables,triggers,uniqueconstraints,views" generateChangeLog```

6. Note that liquibase does not guarantee that every database object will be included in the changelog, you don't have to worry about the most common ones but if there's something too vendor specific you might have to include it manually. That's why it's necessary to double check the generated changelog. See what's missing, create custom scripts to add the missing elements. You can implement the custom stuff by defining it in Java code and then executing with the "customChange" command (https://docs.liquibase.com/change-types/custom-change.html).

7. Create a dbchangelog.xml at the root of the liquibase folder (same name as was declared in liquibase.properties). This will be the root of all the other changelogs in this project. It will include a reference to the dbchangelog.xml of each release plus a reference to the init_changelog.xml (that includes the output of generateChangeLog of the db before liquibase is introduced).

8. Organize changelogs by major-release in a directory structure. A parent folder for every release and a subfolder for every ticket. Every db change that can be defined in a single transaction, should be defined in its own file. Every folder will have a main changelog file for that level. Main changelog file will reference release changelog files and every release changelog file will reference ticket changelog files. Ticket changlog files will reference the ticket related changesets (each changeset in its own file). 

 You should create a separate changeset file for every change that is auto-commited by your database. As a rule of thumb across most database implementations it's the DDL statements that are auto commited. From the liquibase documentation (https://docs.liquibase.com/concepts/changelogs/changeset.html#:~:text=Liquibase%20attempts%20to%20execute%20each,to%20an%20unexpected%20database%20state.):

```Liquibase attempts to execute each changeset in a transaction that is committed at the end, or rolled back if there is an error. Some databases auto-commit statements, which interferes with this transaction setup and could lead to an unexpected database state. Therefore, it is best practice to have just one change per changeset unless there is a group of non-auto-committing changes that you want to apply as a transaction such as inserting data.```

So basically have every DDL statement in it's own changeset, but you can group a number of say INSERT statements into a single changeset so they are applied atomically.

9. The order of execution of the changelogs is explicitly defined within the dbchangelog.xml parent changelogs (in the order they are listed). It is still nice to make the files appear in order of execution on your file system. A solution is to prefix each ticket with the date in yyyyMMdd. If there are more than one created on the same day we can add a _1, _2 etc after the date.
   You can also prefix each file within each feature folder with 01,02 etc (if there's more than one).

10. Might be also nice to prefix the ticket names with "feature" or "bugfix" etc.

11. The changeset ids don't have to have the date prefix since there are other ways to list them in order when browsing table data with sql queries.

12. The last changelog of every ticket and every release should be a tag. The tags will aid in rolling back whole tickets or whole releases.

```liquibase rollback --tag=v0.0.1```

13. Part of the reason to use liquibase is to be able to create a copy of your database in other environments, like qa or dev. You can simply run your changelogs on an empty database. If your changelogs include sensitive data you'll have to create a separate script to sanitize it. 
Unfortunately liquibase does not provide the ability to create an empty new database, you'll have to do that manually.

14. Run the previously generated changelog. Make sure liquibase.properties points to the new db.

```liquibase update```

16. Notice "databasechangelog" table contains one record per changeset. You can add new changesets straight into the main changelog file but you cannot modify existing changesets because that would violate liquibase integrity constraints imposed by stored changeset hashes. This provides you with a clear auditable trail of exactly how the current database evolved to its current state.

17. Every changset that is added after the initial snapshot should have the capacity to be rolled back. This has to be enforced as a practice and verified during code reviews. A lot of things (like simple table creation) don't need an explicit rollback strategy defined. Other ones (like truncating a table) must have the rollback defined explicitly. Simplest rollback command is:

```liquibase rollbackCount 1```
