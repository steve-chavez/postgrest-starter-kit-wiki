## Overview
In the [previous page](Production-Infrastructure) we have setup our production infrastructure and build pipeline (ECS/RDS/Github/CircleCI) and it's just waiting to build our code. As you know, pushing database code to production is not as easy as replacing some files. We need to generate the SQL DDL (Data Definition Language) statements that take our database code from the old version to the new one.
This is done through "migration" files. An excellent tool to manage and track those files is [Sqitch](http://sqitch.org/).
This would be enough if we would have used the database as a "dumb store" since, after the first version is deployed, there would be very rare and few changes to the schema, so writing those files by hand would not be that hard. We, however, have a lot of code in the database that guarantees the integrity of our data. We've also split that code into multiple files and directories. Having to remember what changed and where would be impossible, so this is were the dev tools come in handy again. You do not have to remember what files you changed in order to implement a feature, you change any sql file you need, and at the end, when you have your feature ready to be pushed to production, you just run one command, and that migration file will be generated for you. This comes with one warning though. While those statements will most of the time be exactly what you need to run, you will still have to review them and make sure they do not change something that they should not. This is especially needed when your feature involved changes to the tables holding the data (like renaming a column, adding an index). You can be a bit more relaxed when the changes involve only views and stored procedures since a mistake there can be fixed with a new version.

But anyway, let's see how that process actually goes

## Initial Database State
Right now our code is split into multiple files. We need to create the first migration that has the initial state of our database.

Make sure your stack is up (we need it running when creating migrations), if not, bring it up with `docker-compose up -d`

Create the first migration

```sh
subzero migrations init
```

If all went well, you should see this output
```
Created sqitch.conf
Created sqitch.plan
Created deploy/
Created revert/
Created verify/



Created deploy/0000000001-initial.sql
Created revert/0000000001-initial.sql
Created verify/0000000001-initial.sql
Added "0000000001-initial" to sqitch.plan
```

Tell git to start tracking migration files (just like any other code)

```sh
git add db/migrations/
git commit -m "Initial database state migration"
git push
```

This last push will trigger CircleCI and it will start testing the code again, although it will not push it to production.
Let the process finish and check it's *green*.

## Adding data to migrations
If you look at the `db/migrations/deploy/0000000001-initial.sql` file, you'll see only statements that will create all your entities, you will not see any statement that will add the actual data in the tables that you currently have in your dev stack. For most of the data, this is true, however there are still some places where the data in the tables is more like a configuration than user generated data so it also needs to be reflected in a migration. Since it's impossible to know which is which, this will be a manual step, and also, this is only specific to the starter-kit example, your application might not have that kind of data.

Create an empty migration
```
subzero migrations add --no-diff --note "initial dataset" data

# result
Created deploy/0000000002-data.sql
Created revert/0000000002-data.sql
Created verify/0000000002-data.sql
Added "0000000002-data" to sqitch.plan
```

Change `db/migrations/deploy/0000000002-data.sql` to this

```sql
-- Deploy app:0000000002-data to pg

BEGIN;

SET search_path = settings, pg_catalog, public;

COPY secrets (key, value) FROM stdin;
jwt_lifetime	3600
auth.default-role	webuser
auth.data-schema	data
auth.api-schema	api
\.

INSERT INTO secrets (key, value) VALUES ('jwt_secret', gen_random_uuid());

COMMIT;
```

And now the revert migration `db/migrations/revert/0000000002-data.sql` to this

```sql
-- Revert app:0000000002-data from pg

BEGIN;

SET search_path = settings, pg_catalog;
TRUNCATE secrets;

COMMIT;
```

Remember to also add the migration files to git

```sh
git add db/migrations/
git commit -m "Initial dataset migration"
git push
```


# Incremental changes/migrations
After you've finished work on some feature, you need to creat the migration file. Let's check that process.

Lets make a change to `todos` view.
Change [this line](https://github.com/subzerocloud/postgrest-starter-kit/blob/master/db/src/api/todos.sql#L9)
```sql
#before
select id, todo, private, (owner_id = request.user_id()) as mine from data.todo;

#after
select ('#' || id::text) as id, ('do this: ' || todo) as todo, private, (owner_id = request.user_id()) as mine from data.todo;
```

Save the file (the `subzero dashboard` needs to be allways running when making changes)

Now let's create the migration

```sh
subzero migrations add --note "change todos view" todos

# result
Starting temporary Postgre database
325638e3b549cc6eb41a87097295d6203629459782a2914ccb4fdf4ce3a0594e
Waiting for it to load
PostgreSQL init process complete; ready for start up.

Created deploy/0000000003-todos.sql
Created revert/0000000003-todos.sql
Created verify/0000000003-todos.sql
Added "0000000003-todos" to sqitch.plan


Diffing sql files
ATTENTION: Make sure you check deploy/0000000003-todos.sql for correctness, statement order is not handled!
```

Now open `db/migrations/deploy/0000000003-todos.sql` file and see the result

```sql
START TRANSACTION;

SET search_path = api, pg_catalog;

DROP VIEW todos;

CREATE VIEW todos AS
	SELECT ('#'::text || (todo.id)::text) AS id,
    ('do this: '::text || todo.todo) AS todo,
    todo.private,
    (todo.owner_id = request.user_id()) AS mine
   FROM data.todo;
REVOKE ALL ON TABLE todos FROM webuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE todos TO webuser;
REVOKE ALL(id) ON TABLE todos FROM anonymous;
GRANT SELECT(id) ON TABLE todos TO anonymous;
REVOKE ALL(todo) ON TABLE todos FROM anonymous;
GRANT SELECT(todo) ON TABLE todos TO anonymous;

COMMIT TRANSACTION;
```

The devtools correctly detected that only one view changed since your last migration and created the appropriate DDL statements.

As with any new migration, remember to also add it to git

```sh
git add db/migrations/
git commit -m "Change to the todos view"
git push
```

# Pushing to Production
The step we've all been waiting for.
After you've made sure all tests pass, and all the correct migrations are in git, you will push new code to production by creating a git tag.
CircleCI will take care of building the new docker images, deploying the latest database migrations and tell ECR to start using the new images. 

So the only step here is

```
git tag v0.0.1
git push --tags
```

Now relax and watch CircleCI do the work

