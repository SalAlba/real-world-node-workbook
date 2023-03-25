# Node.js application architecture without magic

In this workshop we'll be building parts of the [Real-World App project](https://github.com/gothinkster/realworld) backend.

You can check the full app with UI and backend here: https://hyperapp.netlify.app/

In this course we'll focus on 3 parts of the app:

### Article

![Article](images/article.png)

### Favorite

![Favorite](images/favorite.png)

### Tags

![Tags](images/tags.png)

There will be 3 main **architectural drivers** in this course:
* **composability** - we want to compose our code like Lego blocks 
* **type-safety** - we want to make full use of modern type systems
* **testability** - we want our code to be easy in any test runner without using hacky techniques like import mocking

## Using minimal scripts and dependencies

Question: Do you think code is an asset or a liability?

There's a popular joke about the size of node_modules of a typical Node.js app. But it doesn't have to be this way.
![Minimal dependencies](images/minimal_deps.png)

Go to **package.json** and check the available scripts. 

Run `npm i --omit=dev` to install all production dependencies that would be needed to build a full version of the Real-World App.
In this course we're not going to use all of them though.

Check the **size of all production deps** with `du -sh node_modules`. As you can see we should cap around 15MB. This is because
we avoid using big frameworks bringing lots of transitive dependencies. 

Every line of [code is a liability](https://wiki.c2.com/?SoftwareAsLiability), and we'll be very picky about **choosing only the things that we need**. 
We don't want to solve framework author's problem but our own. And our strategy will be to
pick small libraries [doing one thing well and playing well with others](https://dannorth.net/2022/02/10/cupid-for-joyful-coding/).

Now run full install `npm i` to get all dependencies, including dev dependencies.

## Getting to know the application through the existing tests

![Testing strategies](images/testing_strategies.png)

Go to [Martin Fowler's bliki](https://martinfowler.com/articles/microservice-testing/#conclusion-summary) to check all types of tests that we
have in a typical backend application. How would you categorize our **src/app.test.ts**?

What do you think about test and production code sitting next to each other? Have you heard of the **common-closure principle**?

**"Things that change together should be kept together"**

## Understanding pipes and filters architecture

Now that you've seen the application behavior described as tests, go to the production code.

To understand lightweight frameworks like `express` we need to unpack the **pipes and filters architecture**

![images/pipes_and_filters.png](images/pipes_and_filters.png)

The interesting part in `express` is how **error handlers are added to the end of the pipeline**. Not-found handler usually goes as second last.
And the generic catch-all handler goes at the very end. Any `next(err)` invocations or errors thrown in previous handlers end up in the 
catch-all handler.

## Extracting routing from the application

![Extracting router](./images/extract_router.png)

Create a new file **src/articlesRouter.ts** and move all the articles related code to the file.
Here's some tips how to get started.

```ts
import {Router} from "express";

export const articlesRouter = Router();

articlesRouter.post("/api/articles", async (req, res, next) => {
});
```

In **src/app.ts** you can include the router like this:
```ts
import {articlesRouter} from "./articlesRouter";

app.use(articlesRouter);
```

Note: it's perfectly fine for smaller apps to stay in one file for a long time. In this course we'd like to learn
different building blocks of a typical backend app. But don't feel obliged to use all of them. Just know what's on the menu.

## Extracting error handlers from the application

![Extracting error handlers](./images/extract_error_handlers.png)

Now it's your turn. Move all the error handling code to **src/errorHandler.ts**.

## Separating domain type and introducing tiny types

![xtracting domain type](./images/extracting_domain.png)

Move the type for `Article` into **src/article.ts**. As a bonus we can add **tiny-types** to improve type-level domain vocabulary.

```ts
export type Tag = string;
export type ArticleId = string;
export type Slug = string;
```

## Test-driving in-memory article repository

![TDD](./images/tdd.png)

Saving articles to a collection was fine for now, but we'd like to introduce an abstraction for storing and retrieving articles. 
We'll call this a **repository**.
Eventually we'll store data in a real database so a good abstraction will come in handy. We also want to prove that you can build
significant portions of your app without making investments into a DB technology. 

Here's a few hints:
* article repository should start with a clean state in each test
* all operations should be async so that it's compatible with a real DB repository
* remove `it.skip` only when you have the previous tests passing 

**src/inMemoryArticleRepository.test.ts**
```ts
import {Article} from "./article";
import {inMemoryArticleRepository} from "./inMemoryArticleRepository";
import assert from "assert";

describe('In memory article repository', function () {
  it('should create articles', async function () {
    const article: Article = {
      id: "id",
      slug: "the-title",
      title: "The title",
      body: "body",
      tagList: ["tag1", "tag2"],
      description: "description",
      createdAt: new Date(),
      updatedAt: new Date()
    };
    const repository = inMemoryArticleRepository();

    await repository.create(article);

    const result = await repository.findBySlug("the-title");

    assert.deepStrictEqual(result, article);
  });

  it.skip('should update articles', async function () {
    const article: Article = {
      id: "id",
      slug: "the-title",
      title: "The title",
      body: "body",
      tagList: ["tag1", "tag2"],
      description: "description",
      createdAt: new Date(),
      updatedAt: new Date()
    };
    const repository = inMemoryArticleRepository();

    await repository.create(article);
    await repository.update({...article, body: 'updated body'});

    const result = await repository.findBySlug("the-title");

    assert.deepStrictEqual(result!.body, 'updated body');
  });

  it.skip('should return null when article not found', async function () {
    const repository = inMemoryArticleRepository();

    const result = await repository.findBySlug("the-title");

    assert.deepStrictEqual(result, null);
  });

});
```

## Replacing variable with repository

![Repository](./images/repository.png)

Switch `articles` variable with the `articleRepository`.

## Introducing application service methods

![Application service](./images/application_service.png)

Our router should not be concerned with making slugs or generating ids. Instead of a typical **application service** object
we can introduce a method for handling article creation **use case/workflow**.

```ts
articlesRouter.post("/api/articles", async (req, res, next) => {
    const input = req.body.article;

    const article = await createArticle(articleRepository, articleIdGenerator)(input);

    res.json({ article: omit(article, "id") });
});
```

To make it easier to implement `createArticle` use case, I prepared a test to drive your production code:

**src/createArticle.test.ts**
```ts
import {createArticle} from "./createArticle";
import assert from "assert";
import {inMemoryArticleRepository} from "./inMemoryArticleRepository";
import omit from "lodash.omit";

describe("Create article", function () {
    it("happy path", async function () {
        const articleRepository = inMemoryArticleRepository();
        const idGenerator = () => "articleId";
        const create = createArticle(articleRepository, idGenerator);

        const article = await create(
            {
                title: "The title",
                body: "body",
                description: "",
                tagList: ["tag1", "tag2"],
            }
        );

        const fetchedArticle = await articleRepository.findBySlug(article.slug);

        assert.deepStrictEqual(omit(fetchedArticle, 'createdAt', 'updatedAt'), {
            body: "body",
            description: "",
            id: "articleId",
            slug: "the-title",
            tagList: ["tag1", "tag2"],
            title: "The title",
        });
    });

});
```
Here's some TS typing hints:
* To get article repository type for now you can use: `ReturnType<typeof inMemoryArticleRepository>`
* Introduce `IdGenerator` type
* Introduce `ArticleInput` type. It should not be the same as the `Article` type.

### Big lesson: curried functions over classes

Most enterprise applications use classes as a main building block. A common pattern is to inject dependencies as constructor arguments
and to pass actual data as method arguments.
```ts
class MyClass {
    constructor(deps) {
    }
    action(data) {}
}
```
A much simpler approach is to **replace classes with curring**.
```ts
const action = (deps) => (data) => {}
```
Every action from your class becomes a separate action with curried dependencies. The more stable dependencies go first.
The more dynamic data goes second.

## Understanding what a unit is in unit test

Unpacking the difference between **unit of code** and **unit of behavior**.

![./images/unit.png](./images/unit.png)

Many people stick to **London/Mockist school of testing**. They test every piece of code as a unit. A class, an object or a function.
To isolate each unit of code from the rest of the codebase they use mocks.
Very often it leads to tests that make refactoring difficult as those tests reflect the structure of the code under test.

What we've just seen in our code is  **a Chicago/Detroit school of testing**. 
Application actions are our unit of behavior under test. This unit of behavior consists of
the application service method talking to in-memory repositories. So a few units of code make up a unit of behavior.
Some people would argue that it's an integration test but since we're not touching any I/O and those tests are extremely fast
it's still a unit test.

For an in-depth comporison of those schools of testing I recommend [this book](https://www.manning.com/books/unit-testing).

## Deciding where to put the repository interface type

![Repository type](./images/repository_type.png)

We'd like to have an explicit type for the `ArticleRepository`. Try to write this type and decide where to put this type definition.

Hints:
* your IDE may have and option to extract existing types
* once you extract a repository type, you can delete explicit typings in the in-memory implementation

## Making dates testable

![Single date](./images/single_date.png)

A good heuristic of a testable codebase is only one occurrence of `new Date()` in the entire codebase.
In the previous test we had to omit `createdAt` and `updatedAt` since they are not testable. We need to fix this.

Create **src/clock.ts**
```ts
export type Clock = () => Date;
export const clock = () => new Date();
```
This is our production implementation of a current date/time. We can swap this for unit testing purposes. 

Update our **src/createArticle.test.ts** and inject a fixed date/time clock so that we can remove `omit`.

## Parsing input data

![Zod](./images/zod.png)

`createArticle` accepts `ArticleInput` type. Check what type we get at the method call site in the `articlesRouter`?

Create **src/parseArticleInput.ts**
```ts
import { z } from "zod";

export const ArticleInput = z.object({
  title: z.string().min(1),
  body: z.string(),
  description: z.string(),
  tagList: z.array(z.string()),
});
export type ArticleInput = z.infer<typeof ArticleInput>;
```

Here's how to use it:
```ts
import {ArticleInput} from "./parseArticleInput";

const input = ArticleInput.parse(req.body.article);
```
What are the types now? 

Go and update `createArticle.ts` to use the `ArticleInput` from `zod`. 

[Parsing libraries are much better than validation libraries](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) because parsers give us **compile time type checking based on runtime schemas**.
In modern software development we **don't have to write those DTOs by hand**.

## Handling parsing errors

![422 Code](./images/422.png)

Try adding this test case to **src/app.test.ts**
```ts
const failedArticle = await createArticle(
    request,
    // @ts-ignore
    {
        title: "",
    },
    422
);

assert.deepStrictEqual(failedArticle.body.errors.length, 4);
```
Our contract for invalid input data will be **422 HTTP status code** and input parsing errors from `zod`.

Try to think how we can handle this error. Where would you put the error handling logic?

## Extracting remaining application service/use case/workflow

Extract `updateArticle` workflow based on what we've learned previously.
The input type for this action should make all fields optional. Only the fields that are present will be updated.

## Starting a database

![PostgreSQL](./images/postgres.png)

Time to persist our articles in a real database.

Here's a helpful `Docker` command to spin up `PostgreSQL` locally:
```dockerfile
docker run -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_USER=user -e POSTGRES_DB=conduit \
  --name conduit -d postgres
```

Try to connect to the DB with a client app (e.g. `pgAdmin`).

## Setting up DB migrations

Create `src/migrate-to-latest.ts` file:

```typescript
import * as path from "path";
import { promises as fs } from "fs";
import {
  Kysely,
  Migrator,
  PostgresDialect,
  FileMigrationProvider,
} from "kysely";
import { Pool } from "pg";

async function migrateToLatest() {
  const db = new Kysely({
    dialect: new PostgresDialect({
      pool: new Pool({
        connectionString: process.env.DATABASE_URL,
      }),
    }),
  });

  const migrator = new Migrator({
    db,
    provider: new FileMigrationProvider({
      fs,
      path,
      migrationFolder: path.join(__dirname, "migrations"),
    }),
  });

  const { error, results } = await migrator.migrateToLatest();

  results?.forEach((it) => {
    if (it.status === "Success") {
      console.log(`migration "${it.migrationName}" was executed successfully`);
    } else if (it.status === "Error") {
      console.error(`failed to execute migration "${it.migrationName}"`);
    }
  });

  if (error) {
    console.error("failed to migrate");
    console.error(error);
    process.exit(1);
  }

  await db.destroy();
}

migrateToLatest();
```

## Creating our first migration

Here's the tables we want to create:
![./images/tables.png](./images/tables.png)

In order to avoid manual DB schema maintenance we'll automate
**database schemas as code**.

Create **src/migrations/0001_create_article.ts**

```typescript
import { Kysely, sql } from "kysely";

export async function up(db: Kysely<any>): Promise<void> {
    await db.schema
        .createTable("article")
        .addColumn("id", "uuid", (col) => col.primaryKey())
        .addColumn("createdAt", "timestamp", (col) => col.notNull())
        .addColumn("updatedAt", "timestamp", (col) => col.notNull())
        .addColumn("slug", "text", (col) => col.notNull())
        .addColumn("title", "text", (col) => col.notNull())
        .addColumn("body", "text", (col) => col.notNull())
        .addColumn("description", "text", (col) => col.notNull())
        .execute();

    await db.schema
        .createTable("tags")
        .addColumn("name", "text", (col) => col.notNull())
        .addColumn("articleId", "uuid", (col) =>
            col.references("article.id").notNull().onDelete("cascade")
        )
        .addPrimaryKeyConstraint("tags_primary_key", ["name", "articleId"])
        .execute();
}

export async function down(db: Kysely<any>): Promise<void> {
    await db.schema.dropTable("article").execute();
    await db.schema.dropTable("tags").execute();
}
```

Finally, apply the migrations:
```
"migrate:latest": "DATABASE_URL=postgres://user:secret@localhost:5432/conduit ts-node src/migrate-to-latest.ts"
```

After running this script we can inspect our DB schema.

## Generating TS types from DB

![DB types](./images/db_types.png)

In order to write type-safe queries we need to **generate DB types from DB schemas**.
Type-safe queries give us confidence that DB changes don't break our code.

```
"generate:types": "DATABASE_URL=postgres://user:secret@localhost:5432/conduit kysely-codegen --out-file=src/dbTypes.ts"
```

Analyze the file that was generated.

## Implementing SQL article repository

![SQL repository](./images/sql_repository.png)

Create **src/sqlArticleRepository.ts** with the following starter code:
```typescript
import {ArticleRepository} from "./article";
import {Kysely} from "kysely";
import {DB} from "./dbTypes";

export const sqlArticleRepository = (db: Kysely<DB>): ArticleRepository => {

  return {
    async create(article) {
    },
    async update(article) {
    },
    async findBySlug(slug) {
    },
  };
};
```
We can live code the implementation together.
Please pay attention to **auto-completion support in the IDE**.

Here's an interesting take on ORMs from the Kysely author who used to work on
a very popular ORMs `Objection.js`:
![ORM vs query builder](./images/orm.png)

## Integration testing SQL repository

![DB integration test](./images/db_integration_test.png)

First we need to connect to our database. For performance reasons
we'll create **a connection pool** of reusable connections instead
of making a new connection per request.

**src/db.ts**
```typescript
import { Kysely, PostgresDialect } from "kysely";
import { DB } from "./dbTypes";
import { Pool } from "pg";

export const createDb = (connectionString: string) =>
    new Kysely<DB>({
        dialect: new PostgresDialect({
            pool: new Pool({
                connectionString,
            }),
        })
    });
```

Copy-paste our in-memory repository test and try to test our new SQL repository against the same contract.

Some hints:
* for SQL variant, `article id` needs to be `UUID` e.g. "200db642-a014-46dc-b678-fc2777b4b301"
* clean DB before each test (```await db.deleteFrom("article").execute();```). Why is it better to clean before?

## Creating contract for a repository

![Repository contract](./images/repository_contract.png)

We had to create almost identical tests twice. 
Try to remove duplication between the repositories by introducing a contract for what it takes to be an article repository. 
This contract should serve us well if we decide to implement another variant e.g. MongoDB.

## Choosing in-memory vs SQL mode

![In-memory vs SQL mode](./images/memory_vs_sql.png)

Personally, I'm a big fan of applications starting in full **in-memory mode by default when no environment variables are provided**.
However, when we have a `DATABASE_URL` we can start the application in the SQL/DB mode.

In the **src/articlesRouter.ts**
```typescript
const articleRepository = process.env.DATABASE_URL ? sqlArticleRepository(createDb(
    process.env.DATABASE_URL
)) : inMemoryArticleRepository();
```

Test the setting by changing **package.json**
```
    "start": "DATABASE_URL=postgres://unleash_user:passord@localhost:5432/articles ts-node src/server.ts",
    "test": "DATABASE_URL=postgres://unleash_user:passord@localhost:5432/articles mocha 'src/**/*.test.ts'",
```

There's a problem in the test command. In the SQL mode the id of article is expected to be `uuid`.

## Injecting real uuid id generator

![Id generator](./images/id_generator.png)

We need a type/interface for a generator and multiple implementations.
This is a similar technique to a repository type/interface and multiple implementations.

Create **src/uuidGenerator.ts**
```typescript
import { v4 as uuidv4 } from "uuid";
import { IdGenerator } from "./idGenerator";

export const uuidGenerator: IdGenerator = uuidv4;
```
Inject this generator in **src/articlesRouter.ts** depending on the `DATABASE_URL` config.
Only SQL/DB mode should use this generator.

## Reading env vars from a file

![dotenv library](./images/dotenv.png)

Currently, we're updating our **package.json** scripts with the env vars. 
If we want to add more env vars this approach will not scale.

Create **.env** in a root of the project:
```
PORT=3000
DATABASE_URL=postgres://user:secret@localhost:5432/conduit
```

Modify your scripts:
```
    "start": "ts-node -r dotenv/config src/server.ts",
    "test": "mocha 'src/**/*.test.ts'",
```

[-r preload option](https://nodejs.org/api/cli.html#-r---require-module) allows to run `dotenv` module before our application starts. 
It sets all the env vars upfront. 

You can also inspect **.mocharc.js** to see how `mocha` allows to preload the env vars from a file.
For `mocha` I prefer to have a configuration file instead of `-r`.
I run my tests both from the CLI and from my IDE and both should respect the
configuration file.

## Modelling type-safe config

![Type-safe config](./images/safe_config.png)

Currently, `process.env` is spread across the entire codebase. We'd like to centralize **config setting at the entry point to our application**
since config will be used everywhere. As an extra benefit we'd like to **parse the config** at the startup time the same way we 
parse request body in our controllers/routers.

Create **src/config.ts**
```typescript
import { z } from "zod";

const Config = z.object({
    PORT: z.coerce.number(),
    DATABASE_URL: z.string().optional(),
});
export type Config = z.infer<typeof Config>;

export const config = Config.parse(process.env);
```

## Injecting config to application and router

![Config injection to application](./images/inject_config.png)

Read the config in the **src/server.ts** and pass it to other places that need it.

```typescript
import { createApp } from "./app";
import { config } from "./config";

const port = config.PORT;
const app = createApp(config);
app.listen(port, () => {
  console.log(`Listening on ${port}`);
});
```
Change your code, so you can easily inject the config to your application.
Application should be able to inject the config to the router.
Remember about changing our component test so that it respects the injection of config.

## Introducing a use case composition root

![Composition root](./images/composition_root.png)

**src/articleRouter.ts** has 2 responsibilities. It sets up the routes/controllers but also builds a graph of objects.
Split the graph of objects into a [composition root](https://blog.ploeh.dk/2011/07/28/CompositionRoot/).
The composition root will assemble all objects and functions to have
fully functional article actions either in SQL or in-memory mode.

Here's the structure we want to have in **src/app.ts**
```typescript
const db = config.DATABASE_URL ? createDb(config.DATABASE_URL) : null;
const articleActions = db
    ? sqlArticlesCompositionRoot(db)
    : inMemoryArticlesCompositionRoot();
app.use(createArticlesRouter(articleActions));
```

And the actual router should accept all the actions and repository as dependencies:
```typescript
export const createArticlesRouter = ({
  create,
  update,
  articleRepository,
}: {
  create: CreateArticle;
  update: UpdateArticle;
  articleRepository: ArticleRepository;
}) => {
    // const article = await create(input);
    // const article = await update(slug, articleInput);
    // const existingArticle = await articleRepository.findBySlug(slug);
};
```

## Organizing code into directories: feature-driven architecture

![Technology-driven architecture](./images/technology_driven.png)

Technology driven-architectures lead to the [shotgun surgery](https://refactoring.guru/smells/shotgun-surgery) anti-pattern.

![Shotgun surgery](./images/shotgun.png)

Adding/deleting a feature requires touching many unrelated directories.
**Deleting a feature should be just a deletion of a directory**.

I like to organize my **code around business capabilities first** and around technical responsibilities second.

Create **articles** directory and put everything related to article there.

Then put all the shared code into **shared** dir and all error handling related code in **error**.


## Organizing code into directories: technology-driven architecture

![Technology-driven inside business-driven](./images/driven.png)

Once we're done with the feature-driven organization we can go inside article dir and organize code around technical
responsibilities:
* **api**
* **application**
* **domain**
* **infrastructure**

## Cleaning data between component tests

![Shared test data](./images/shared_test_data.png)

If you run our component test separately from the other tests, it will accumulate data from the previous test runs.

Where would you clean the test data between the test?

## Extracting application composition root

![Composition root](./images/composition_root.png)

Our **src/app.ts** is accumulating logic for building dependencies at the high level. 

Extract those into another composition root.  

Note: it's a common misconception that we can only have one composition root. But for
most non-trivial apps it's typical to have **a composition root per module** and then some
**root-level composition root** one level above.

## Introducing transactions

![Transaction boundary](./images/transaction_boundary.png)

Our SQL code is adding data into two database tables: `article` and `tags`.
We would like to avoid storing only the article without storing the tags.
We can group all DB operations as one unit of work called a **transaction**. 

Typically, we put our **transaction boundaries around the application service**.
Some less correct options are:
* putting DB transactions into a repository, but then our transaction can't span across different repositories
* putting DB transactions into the application service itself, but it doesn't make sense for the in-memory variant and also pollutes the application service itself

Create a file **src/shared/sqlTransaction.ts**
```typescript
import { Kysely, Transaction } from "kysely";

export type WithTx<D> = ReturnType<typeof transactional<D>>;

export type AsyncServiceMethod<A extends any[], R> = (...args: A) => Promise<R>;

export const transactional =
    <D>(db: Kysely<D>) =>
        <A extends any[], R>(fn: (tx: Transaction<D>) => AsyncServiceMethod<A, R>) =>
            (...args: A) => {
                return db.transaction().execute((trx) => fn(trx)(...args));
            };
```
To make it easier to read here's a JS version:
```javascript
export const transactional =
  (db) =>
  (fn) =>
  (...args) => {
    return db.transaction().execute((trx) => fn(trx)(...args));
  };
```
* `db` is a regular DB instance we use to get a current transaction
* `fn` is a factory function that accepts a current DB transaction and returns an application service. Since an application service can be any async function we need to accept any number of args and return async value
* the last function is a function that is a decorator for the original application service method. It accepts any number of args to
be a decorator for any application service method. Inside the body it creates a new request-scope transaction on each invocation, injects the transaction
to the factory `fn` function and calls the result with the args.

## Using transactions

Currently, in our system every application service method is a singleton and is created once at the application startup time.

```typescript
  const articleRepository = sqlArticleRepository(db);
  const create = createArticle(articleRepository, articleIdGenerator, clock);
  const update = updateArticle(articleRepository, clock);
```
First, we construct a singleton article repository that we inject to get a singleton create and update methods.

To support transactions we need to create a new mini graph of objects for each application service method.

![./images/transactions.png](./images/transactions.png)

Go to **src/appCompositionRoot.ts** and inject a transaction builder:
```typescript
  const articleActions = db
    ? sqlArticlesCompositionRoot(db, transactional(db))
    : inMemoryArticlesCompositionRoot();
```

Then go to **src/articles/application/articlesCompositionRoot.ts** and change a signature to accept transactions:
```typescript
const create = (db: Transaction<DB>) => {
  const articleRepository = sqlArticleRepository(db);

  return createArticle(articleRepository, uuidGenerator, clock);
};

const update = (db: Transaction<DB>) => {
  const articleRepository = sqlArticleRepository(db);

  return updateArticle(articleRepository, clock);
};
```
`create` and `update` are not singletons anymore, but **factory functions** that accept a current **request-scoped transaction** and
return **transactional service methods**.

Update the usage of those functions:
```typescript
export const sqlArticlesCompositionRoot = (
  db: Kysely<DB>,
  withTxDb: WithTx<DB>
) => {
  const articleRepository = sqlArticleRepository(db);

  return {
    create: withTxDb(create),
    update: withTxDb(update),
    articleRepository,
  };
};
```
`withTxDb` will create a request scope for each invocation of `create/update` and inject a new transaction on each request.
After passing current `tx` to the factory functions we'll get transactional `create` and `update`.

## Deciding where to put favorites count

![Favorite](images/favorite.png)

If you check https://hyperapp.netlify.app/ there's a favorite/unfavorite icon. In our  simplified version we don't track
who liked the article, just the total count. 

Where would you store the total count of favorites for an article?
What DB schema modifictions would you make?
Where would you put it in our model in code?

As you're deciding where to put favorite a trainer will provider some code.

## Introducing internal read model

![Write model vs read model](./images/read_model.png)

In a typical web app we can separate two models:
* write model 
* read model 

Read model usually serves the UI display and is often called a view model. 
But read model can also be used to serve other modules. 

Favorites implementation is using article repository to check if the article exists. But it couples the favorites module
to the entire repository implementation. Favorites module is only interested in one question:
does the article with a given slug exist and what's the id?

Create SQL and in-memory read model matching the following type in **src/article/infrastructure/articleReadModel.ts**
```typescript
import { ArticleId, Slug } from "../domain/article";

export type ArticleReadModel = {
  findArticleIdBySlug(slug: Slug): Promise<ArticleId | null>;
};
```
Hint:
* in-memory read model will need to read data off the repository

The best treatment of read models I found comes from [this book](https://matthiasnoback.nl/book/advanced-web-application-architecture/).

## Replacing repository with read model in favorites

![Article read model](./images/article_read_model.png)

In the favorites module replace the dependency on the article repository with the article read model.
Run your tests to verify if everything works both with SQL and in-memory variants.

## Implementing view model (external read model)

Add `favoritesCount` file in **app.test.ts**: 
```typescript
    assert.deepStrictEqual(
      omit(articleResult.body.article, "createdAt", "updatedAt"),
      {
        body: "body",
        description: "description",
        tagList: ["tag1", "tag2"],
        title: "The title",
        slug: "the-title",
        favoritesCount: 0,
      }
    );
```
When asking about the article we should show data from the article and favorite modules.

Here's the expected read model type in **src/article/infrastructure/articleReadModel.ts**
```typescript
import { Article, ArticleId, Slug } from "../domain/article";

export type ArticleView = {
  article: Omit<Article, "id"> & { favoritesCount: number };
};

export type ArticleReadModel = {
  findArticleIdBySlug(slug: Slug): Promise<ArticleId | null>;
  findArticleBySlug(slug: Slug): Promise<ArticleView | null>;
};
```

Start by implementing the SQL repository. You can also try to implement the in-memory version.
Together we'll live code the solution for plugging it into our composition roots.

## Parsing output data

![Robustness principle](./images/robustness.png)

Our previous view/read model returns too much data. We effectively break the [robustness principle](https://en.wikipedia.org/wiki/Robustness_principle) by returning
extra fields that are not part of the contract. 
While we can manually hand-pick the DB columns we need, there's
a more robust way to do it.

Replace our handwritten `ArticleView` with a `zod` type.

## Post/Redirect/Get pattern

![Post/Redirect/Get Pattern](./images/prg.png)

We'd like every POST and PUT operation we have (article create/update, favorite/unfavorite) to return full article view
with the favorites count. There's a cheap way to implement it. We can **redirect after every POST/PUT operation**.

```typescript
res.redirect(`/api/articles/${slug}`);
```
This will set the **HTTP code to 302** and set the **Location header** to the new resource the client can GET. Most HTTP clients can
follow the 302 response and read the Location header automatically before making another request.

Go to **app.test.ts** and fix all operations that should redirect:
```typescript
   const createdArticle = await createArticle(request, {
      ...
    }).redirects(1);
```

## Type-safe paths

![Type-safe paths](./images/type_safe_paths.png)

In a typical backend system we have routes and links to those routes.
In the previous exercise we created a few links for redirects. Those are not type safe and very error-prone.
It would be nice to link routes and links in a type-safe manner so we can
statically prevent 404s. 

Type-safe paths to the rescue:

Create **src/article/api/paths.ts**
```typescript
import { path } from "static-path";

export const articlePath = path("/api/articles/:slug");
```

You can use it like this:
```typescript
app.use(articlePath.pattern, handler);

res.redirect(articlesPath({slug: "myslug"}));
```

Now we can prevent 404s at compile time.

We can also update remaining links e.g. in tests.

## Creating toggle client abstraction

![Feature toggle](./images/toggle.png)

We'd like to have a capability to **turn the features on and off from config**. This is what **feature toggles** are for.

In the `createApp` we'd like to pass a list of features we want to turn on e.g.
```typescript
createApp(appConfig, ["create"])
```
**src/shared/toggleClient.ts**
```typescript
export type ToggleClient = {
  isEnabled(feature: string): boolean;
};
```

**src/shared/inMemoryToggleClient.ts**
```typescript
import { ToggleClient } from "./toggleClient";

export const createInMemoryToggleClient = (
  enabledFeatures: string[]
): ToggleClient => {
  return {
    isEnabled(feature: string): boolean {
      return enabledFeatures.includes(feature);
    },
  };
};
```

Inject the toggle client into `articlesRoutes` so that we can control which features are enabled.

## Integrating with other services over HTTP

![External toggle service](./images/toggle_service.png)

For more complicated scenarios it'd be nice to control available features from an **external toggle client**.
It allows to **switch the system behavior instantly without a redeployment**.

Create real toggle client at **src/shared/realToggleClient.ts**:
```typescript
import { Unleash } from "unleash-client";
import { ToggleClient } from "./toggleClient";

export const createUnleashToggleClient = async (
    url: string,
    clientKey: string
): Promise<ToggleClient> => {
    const unleash = new Unleash({
        url,
        appName: "real-world-node",
        customHeaders: {
            Authorization: clientKey,
        },
    });
    await unleash.start();
    return unleash;
};
```

Here's the credentials you can use:
```
TOGGLE_API=https://app.unleash-hosted.com/demo/api/
TOGGLE_CLIENT_KEY=56907a2fa53c1d16101d509a10b78e36190b0f918d9f122d
```

Write an integration test verifying if flag called `create` is enabled.
A trainer will turn the flag on in the meantime.

## Injecting asynchronous dependencies

![Async deps and color coding](./images/async_deps.png)

Before our toggle client is ready it needs to fetch toggles configuration over HTTP. It's not instant. 
How would you **inject those asynchronous dependencies into out dependency graph**?

## Avoiding N+1 queries

![N + 1 query problem](./images/n_queries.png)

Before our next exercise we'll need to add a new capability to our repository.

We have a new contract:
```typescript
    it("should return all articles", async function () {
      const article1: Article = {
        id: "200db642-a014-46dc-b678-fc2777b4b302",
        slug: "the-title-1",
        title: "The title 1",
        body: "body",
        tagList: ["tag1", "tag2"],
        description: "description",
        createdAt: new Date(),
        updatedAt: new Date(),
      };
      const article2: Article = {
        id: "300db642-a014-46dc-b678-fc2777b4b302",
        slug: "the-title-2",
        title: "The title 2",
        body: "body",
        tagList: ["tag1", "tag2"],
        description: "description",
        createdAt: new Date(),
        updatedAt: new Date(),
      };
      const repository = articleRepositoryFactory();

      await repository.create(article1);
      await repository.create(article2);

      const result = await repository.getAll();

      assert.deepStrictEqual(result, [article1, article2]);
    });
```
Implement in-memory and SQL versions. Be careful not to run too many
DB queries. I recommend doing it with max 2 queries.

## Doing feature-flag driven development

![Feature-flag driven development](./images/ffdd.png)

Implement GET `/api/tags` that returns all tags from all articles. Keep it behind a feature flag called `articleTags`.
We don't know the implications of the DB query performance, and we want to roll it out to 1% of our users.
```typescript
  it("Get all tags", async function () {
    const toggleClient = await createToggleClient({} as Config, [
      "articleTags",
    ]);
    const { app, clean } = createApp(config, toggleClient);
    const request = httpClient(app);
    await clean();

    await createArticleWithTags(request, ["tag1", "tag2"], 302);
    await createArticleWithTags(request, ["tag2", "tag3"], 302);
    await createArticleWithTags(request, ["tag4", "tag5"], 302);

    const tags = await getTags(request);

    assert.deepStrictEqual(tags.body.tags, [
      "tag1",
      "tag2",
      "tag3",
      "tag4",
      "tag5",
    ]);
  });

```

## Targeting who can see the feature with extra constraints

![Constraints in feature flags](./images/constraints.png)

The real power of feature toggles comes from the advanced rollout strategies. Deciding who can see our new features
at the very granular level.

For an example let's see how we could enable our new tags only for users in Poland.

This is just a tip of the iceberg of what feature-flags can do. Testing in production is
a really powerful concept for developers (gradual rollout to selected internal users), QA (testing different combinations of features on demand), ops (controlling misbehaving features) and even marketing people (a/b tests).

## Making tags unique

There's a tiny bug in our code. You can reproduce it by adding two identical tags in our component test:
```typescript
await createArticleWithTags(request, ["tag4", "tag4"], 302);
```

Try to fix this bug by choosing where to remove the duplication.