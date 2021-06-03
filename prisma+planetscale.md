# Prisma + PlanetScale










Much better version of this at [https://github.com/prisma/prisma/issues/7292](https://github.com/prisma/prisma/issues/7292)














## PlanetScale?

[PlanetScale](https://planetscale.com) recently launched as "the database for developers" and made quite a splash on [Hacker News](https://news.ycombinator.com/item?id=27197873) and [ProductHunt](https://www.producthunt.com/posts/planetscale). Their name obviously is a play on the term "web scale", that describes the scale that big companies like Google, Amazon or Facebook achieve - so quite a claim and promise to live up to.

Quick overview what PlanetScale promises:

- Close to instant provisioning (via UI, CLI and API)
- [Branching of databases](https://docs.planetscale.com/concepts/branching) similar to git
- [Schema management workflow](https://docs.planetscale.com/concepts/nonblocking-schema-changes)  with "deploy requets" (similar to GitHub's Pull Requests)

At they same time, the [pricing model introduced by them](https://www.planetscale.com/pricing) is also quite nice for a database product:

- awesome free tier (3 databases, and a _lot_ of storage, reads and writes)
- if you exceed that, the real pricing is simple and usage based

We like to see innovation like this at Prisma, and we expect our users to be interested in this as well. So:

## How do Prisma and PlanetScale work together?

TODO Write introduction on challenges, include list with anchors to individual sections so it is easy to get overview

<!--
At Prisma we like new databases that do that - and of course jump on them and see how they work with Prisma

We actually have been working on support for Vitess, the underlying MySQL variant, for quite some time, so the "MySQL compatible" of PlanetScale should also apply to our `mysql` connector in Prisma.

While the Pricing just needs to be undestdoof, the novel features that Planetscale introduces create some challenges for us:
... but the product Planetscale released has some properties I thought would be worth explaining.

Let's look at these new challenges to get Prisma and PlanetScale to work together:

-->

### No connection string

When you look at the admin UI of Planetscale, you quickly notice that they do not just provide you with a connection string to connect to your database (as many other database providers do). But you need that to connect to the database and configure your Prisma Schema file!?

PlanetScale abstracts the connection string away for convenience and security reasons. You can use their CLI to create a local tunnel to your database hosted on PlanetScale, and then use the provided local connection string in Prisma. You can either wrap your app into the call to the CLI (e.g. `pscale connect my-database main --execute-protocol 'mysql' --execute 'yarn start'`) or do so by running the CLI in the background and then starting your app in parallel (use `mysql://root:@127.0.0.1:3306/my-database` as the connection string). 

Both work equally fine, but you might prefer the latter, as it also allows you to use the Prisma CLI (e.g. for Migrations) or you will also need to wrap these calls into a `pscale` execution.

After getting used to it, this is actually quite nice workflow as you can seamlessly switch between databases and branches by adapting that `pscale` command.

TODO: How to do this on serverless? (https://prisma.slack.com/archives/CA491RJH0/p1621835460092100?thread_ts=1621384194.012100&cid=CA491RJH0)

### No schema changes on the production branch, or: How PlanetScale's branching system works

TODO Refine draft below

Part of PlanetScale's branching concept is that the production branch is protected from schema changes. Carelessly changing the schema on a production database is a good way to produce downtime, and you usually should not do it - but often there is no "good" way to avoid it. PlanetScale offers that with their branching system.

Prisma currently does not have built in support for Planetscale's branching system, but it is easy to use if you first understood it and follow the same steps each time:

The first step here definitely is to really understand the new branch workflow. As it is a totally new thing, it took me personally a bit to realize that I could not make any schema changes on `main` and I would get the following error message instead:
```
TODO
```

This is how you work with their branching system instead:
1. Create a branch, e.g. `foo` from your current `main`
2. Connect to that branch
3. Run `prisma migrate` there
4. The only thing you have to do manually, is to copy over the entry from the `_prisma_migrations` table after you merged a deploy request that put the schema changes from the development branch into the production branch. 

If you do not do that, you will get an error message like this:
```
C:\Users\Jan\Documents\throwaway\planetscale>npx prisma migrate dev
Environment variables loaded from .env
Prisma schema loaded from prisma\schema.prisma
Datasource "db": MySQL database "fk-test" at "127.0.0.1:3306"

? Drift detected: Your database schema is not in sync with your migration history.      

We need to reset the MySQL database "fk-test" at "127.0.0.1:3306".
Do you want to continue? All data will be lost. » (y/N)
```
To copy over, run the follow commands

```
TODO
```

We are in contact with Planetscale, for them to add this to their product, so you do not have to manually do that each time.

### No shadow database creation

PlanetScale also does not let you execute a `CREATE DATABASE` statement to just create a new database - you are supposed to use their web UI or CLI to create additional databases, or branches on existing databases. This does not match well with Prisma's `migrate dev` that tries to create a [shadow database](https://www.prisma.io/docs/concepts/components/prisma-migrate/shadow-database) to detect drift and create your migrations.

Fortunately [Prisma Migrate already supports environments with this limitation](https://www.prisma.io/docs/concepts/components/prisma-migrate/shadow-database#cloud-hosted-shadow-databases-must-be-created-manually) and you can set a `shadowDatabaseUrl` in the `datasource` block of your `schema.prisma` file to supply a precreated database to be used as the shadow database.

You can either set the connection URL of _any_ MySQL 8 database as the `shadowDatabaseUrl`, but also of course create a dedicated branch on your PlanetScale database via UI or CLI. I suggest you call it `prisma-shadow-database`. Then you can open a connection via `pscale connect my-database prisma-shadow-database --port 3333` (note the custom port) and add `shadowDatabaseUrl = "mysql://root:@localhost:3333/my-database"` to the `datasource` block in your `schema.prisma`. Prisma will clean up that database before each usage, so you can just let it hang around forever.

TODO: Confirm this _really_ works as described.

### No foreign keys

TODO Refine draft below

- The second super interesting concept they apply are Non-Blocking Schema Changes. 
- Do not support foreign keys for that
- Reasoning: http://code.openark.org/blog/mysql/the-problem-with-mysql-foreign-key-constraints-in-online-schema-changes
- Prisma Migrate does not play well with databases that do not support foreign keys.

The missing support for foreign keys is the bigger problem, as Prisma by default relies on them to identify and manage relations between models. Fortunately, a current quirk of Prisma comes in handy here and gives us a great workaround:

Prisma creates foreign keys for all the relations in your Prisma schema.

When you are ready to migrate your database, you switch into a new branch from Planetscale. Then you usually would run `npx prisma migrate dev` to create the migration and apply it. If you added a foreign key, this would now fail with `.... TODO error msg ...`. So instead you run `npx prisma migrate dev --create-only` which will create a new folder in `migrations` with a `migration.sql` file. Open that file and remove all the `ALTER TABLE` statements that create foreign keys. Afterwards you can deploy the modified migration with `npx prisma migrate deploy`.

Note: Any future `prisma migrate dev` will try to create these foreign keys again, so you need to be vigilant in removing these or applying your migration file will fail.

(Do not forget to copy over the new row from `_prisma_migrations` as described above as well to ensure `migrate dev` knows about the already applied migrations.)

Prisma will probably introduct a special mode to support this better.


## Summary

- Use the `pscale connect` CLI to open a connection to the database and use that connection string in Prisma
- Use PlanetScale branches to run `prisma migrate` commands
- Copy over content of `_prisma_migrations` manually after merging a development branch into `main`. Planetscale will improve this in their product soon.
- Use a PlanetScale branch as your shadow database as well
- Remove foreign keys manually from the `migration.sql` generated by `prisma migrate dev --cerate-only`, then use `prisma migrate deploy` to deploy. Prisma will improve this soon by introducing a mode that does this automatically.
