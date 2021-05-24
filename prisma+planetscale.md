# Prisma + PlanetScale

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

TODO

At Prisma we like new databases that do that - and of course jump on them and see how they work with Prisma

We actually have been working on support for Vitess, the underlying MySQL variant, for quite some time, so the "MySQL compatible" of PlanetScale should also apply to our `mysql` connector in Prisma.

While the Pricing just needs to be understand, the novel features that Planetscale introduces create some challenges for us:
... but the product Planetscale released has some properties I thought would be worth explaining.

TODO

Let's look at these new challenges to get Prisma and PlanetScale to work together:

### No connection string

When you look at the admin UI of Planetscale, you quickly notice that they do not just provide you with a connection string to connect to your database (as many other database providers do). But you need that to connect to the database and configure your Prisma Schema file!?

PlanetScale abstracts the connection string away for convenience and security reasons. You can use their CLI to create a local tunnel to your database hosted on PlanetScale, and then use the provided local connection string in Prisma. You can either wrap your app into the call to the CLI (e.g. `pscale connect my-database main --execute-protocol 'mysql' --execute 'yarn start'`) or do so by running the CLI in the background and then starting your app in parallel (use `mysql://root:127.0.0.1:3306/my-database` as the connection string). 

Both work equally fine, but you might prefer the latter, as it also allows you to use the Prisma CLI (e.g. for Migrations) or you will also need to wrap these calls into a `pscale` execution.

After getting used to it, this is actually quite nice workflow as you can seamlessly switch between databases and branches by adapting that `pscale` command.

TODO: How to do this on serverless?

### No schema changes on the production branch

Part of their branching concept is that the production branch is protected from schema changes. Carelessly changing the schema on a production database is a good way to produce downtime, and you usually should not do it - but often there is no "good" way to avoid it.

### No shadow database creation
Planetscale does not let you `CREATE DATABASE` - you should use their web UI to create additional databases, or branches on existing databases. This does not match well with Prisma's `migrate dev` that tries to create a "shadow database" to test run your migrations. 
Just create a branch `shadow-database` via the UI and open a connection to that via `pscale connect foo shadow --port 3333` before you use any `prisma migrate` commands. Use that as the shadow database by adding `shadowDatabaseUrl = "mysql://root:@localhost:3333/foo"` to your `datasource` block. 

### No foreign keys
The second super interesting concept they apply are Non-Blocking Schema Changes. 

Fortunately, the two final challenges also can be solved or worked around, until we bake in better support into Prisma itself:

## Solutions

Prisma currently does not have built in support for Planetscale's branching system, and also does not play well with databases that do not support foreign keys.

### The branching system
The first step here definitely is to really understand the new branch workflow. As it is a totally new thing, it took me personally a bit to realize that I could not make any schema changes on 
This is how you do it:
Create a branch, e.g. `foo` from your current `main`
Connect to that branch
The only thing you have to do manually, is to _copy over the entry from the `_prisma_migrations` table after you merged a deploy request that put the schema changes from the development branch into the production branch. 

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

TODO

We are in contact with Planetscale, for them to add this to their product, so you do not have to manually do that each time.

### No foreign keys

The missing support for foreign keys is the bigger problem, as Prisma by default relies on them to identify and manage relations between models. Fortunately, a current quirk of Prisma comes in handy here and gives us a great workaround:

Prisma creates foreign keys for all the relations in your Prisma schema. It 

When you are ready to migrate your database, you switch into a new branch from Planetscale. Then you usually would run `npx prisma migrate dev` to create the migration and apply it. If you added a foreign key, this would now fail with `.... TODO error msg ...`. So instead you run `npx prisma migrate dev --create-only` which will create a new folder in `migrations` with a `migration.sql` file. Open that file and remove all the `ALTER TABLE` statements that create foreign keys. Afterwards you can deploy the modified migration with `npx prisma migrate deploy`.

(Do not forget to copy over the new row from `_prisma_migrations` as described above.)



## Summary
- Use branches to run `prisma migrate` commands
- TODO shadow database
- Remove foreign keys manually from the `migration.sql` generated by `prisma migrate dev --cerate-only`, then use `prisma migrate deploy` to deploy. Prisma will improve this soon by introducing a mode that does this automatically.
Copy over content of `_prisma_migrations` manually after merging a development branch into `main`. Planetscale will improve this in their product soon.


