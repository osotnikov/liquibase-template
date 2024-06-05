1. Start db

2. Check db up

```psql -U cdl_user -h localhost cdl_control```

3. Make sure liquibase.properties and postgres driver jar are in
   liquibase folder

4. Create init changelog (run generateChangeLog, also referred to as creating a "snapshot" of the db). Probably good to use the default changelog file name (dbchangelog.xml) so other liquibase commands detect it automatically. Include all possible diff types including table data:

```liquibase --changeLogFile=dbchangelog.xml --diff-types="data,catalogs,checkconstraints,columns,databasepackage,databasepackagebody,foreignkeys,functions,indexes,primarykeys,sequences,storedprocedures,tables,triggers,uniqueconstraints,views" generateChangeLog```

5. Create a new db. Liquibase cannot do that automatically for you. You either need to include an extra changeset or create an empty db/schema manually.

6. Run the previously generated changelog. Make sure liquibase.properties points to the new db.

```liquibase update```

7. Compare the existing database with the created one. Liquibase makes best effort when running "generateChangeLog" to generate most constructs, it may have trouble with vendor specific things though.
   See what's missing, create custom scripts to add the missing elements. You can implement the custom stuff by defining it in Java code and then executing with the "customChange" command.

8. Notice "databasechangelog" table contains one record per changeset. You can add new changesets straight into the main changelog file but you cannot modify existing changesets because that would violate liquibase integrity constraints imposed by stored changeset hashes.

9. Every changset that is added after the initial snapshot should have the capacity to be rolled back. We'll have to enforce is as a practice and verify during code reviews. A lot of things (like simple table creation) don't need an explicity rollback strategy defined. Other ones (like truncating a table) must have the rollback defined explicitly. Simplest rollback command is:

```liquibase rollbackCount 1```

CDL_APPS specific organization

1. Have dbchangelog.xml at the root of the liquibase folder. This will be the root of all the other changelogs in this project. It will include a reference to the dbchangelog.xml of each release plus a reference to the init_changelog.xml (that inlcludes the output of generateChangeLog of the db before liquibase is introduced).

2. Organize changelogs by major-release in a directory structure. A parent folder for every release and a subfolder for every ticket. Every db change that is part of a ticket that can be defined in a single transaction, should be defined in its own file. Every folder will have a main changelog file for that level. Main changelog file will reference release changelog files and every release changelog file will reference ticket changelog files. Ticket changlog files will reference the ticket related changesets (each changeset in its own file).

3. The order of execution of the changelogs is explicitly defined by the dbchangelog.xml parent changelogs. It is still nice to make the files appear in order of execution though so I propose we prefix each ticket with the date in yyyyMMdd so they appear in order of execution within each release. If there are more than one on the same day we can add a _1, _2 etc after the date.
   Prefix each file within each feature folder with 01,02 etc (if there's more than one).

4. Might be also nice to prefix the ticket names with "feature" or "bugfix" etc.

5. The changeset ids don't have to have the date prefix since there are other ways to list them in order when browsing table data.

6. The last changelog of every ticket and every release should be a tag. The tags will aid in rolling back whole tickets or whole releases.

```liquibase rollback --tag=v0.0.1```