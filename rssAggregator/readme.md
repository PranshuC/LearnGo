# Welcome

## What are we building?
We're going to build an [RSS](https://en.wikipedia.org/wiki/RSS) feed aggregator in Go! It's a web server that allows clients to:
* Add RSS feeds to be collected
* Follow and unfollow RSS feeds that other users have added
* Fetch all of the latest posts from the RSS feeds they follow

RSS feeds are a way for websites to publish updates to their content. You can use this project to keep up with your favorite blogs, news sites, podcasts, and more!

Reference : https://github.com/bootdotdev/fcc-learn-golang-assets/tree/main/project

## Learning goals
* Learn how to integrate a Go server with PostgreSQL
* Learn about the basics of database migrations
* Learn about long-running service workers

## Setup
Before we dive into the project, let's make sure you have everything you'll need on your machine.
1. The latest [Go toolchain](https://golang.org/doc/install).
2. [VS code](https://code.visualstudio.com/) editor and official [Go extension](https://marketplace.visualstudio.com/items?itemName=golang.Go).
3. [Thunder Client](https://www.thunderclient.io/) HTTP client.

### Install packages
Install the following packages using `go get`:
* [chi](https://github.com/go-chi/chi/v5)
* [cors](https://github.com/go-chi/cors)
* [godotenv](github.com/joho/godotenv)
* [uuid](github.com/google/uuid)
* [pq](github.com/lib/pq)

### Env
Create a gitignore'd `.env` file in the root of your project and add the following:
```bash
PORT="8080"
```

## PostgreSQL
PostgreSQL is a production-ready, open-source database.  Postgres is itself a server and listens for requests on default port `:5432`.

To interact with Postgres, first you will install the server and start it. Then, you can connect to it using a client like [psql](https://www.postgresql.org/docs/current/app-psql.html#:~:text=psql%20is%20a%20terminal%2Dbased,or%20from%20command%20line%20arguments.) or [PGAdmin](https://www.pgadmin.org/).

### 1. Mac OS : Install
[brew](https://brew.sh/) recommended :
```bash
brew install postgresql
```

### 2. Ensure the installation worked
The `psql` command-line utility is the default client for Postgres. Use it to make sure you're on version 14+ of Postgres:
```bash
psql --version
```

### 3. Mac OS : Start the Postgres server in the background
```bash
brew services start postgresql
```

### 4. Mac OS : Connect to the server
Open PGAdmin client and create a new server connection. Here are the connection details you'll need (with brew) :
* Host: `localhost`
* Port: `5432`
* Username: Your Mac OS username
* Password: *leave this blank*

### 5. Create database and Test
In PGAdmin's dropdown menu on the left, open the `Localhost` tab, then right click on "databases" and select "create database". Name it and remember for future.

Right click on your database's name in the menu on the left, then select "query tool". In the text editor, type the following query:
```sql
SELECT version();
```

## Create Users
We'll be adding an endpoint to create new users on the server using a couple of tools :
* [database/sql](https://pkg.go.dev/database/sql): This is part of Go's standard library. It provides a way to connect to a SQL database, execute queries, and scan the results into Go types.
* [sqlc](https://sqlc.dev/): SQLC is an *amazing* Go program that generates Go code from SQL queries. It's not exactly an [ORM](https://www.freecodecamp.org/news/what-is-an-orm-the-meaning-of-object-relational-mapping-database-tools/), but rather a tool that makes working with raw SQL almost as easy as using an ORM.
* [Goose](https://github.com/pressly/goose): Goose is a database migration tool written in Go. It runs migrations from the same SQL files that SQLC uses, making the pair of tools a perfect fit.

### 1. Install SQLC
SQLC is just a command line tool, it's not a package that we need to import. I recommend [installing](https://docs.sqlc.dev/en/latest/overview/install.html) it using `go install`:
```bash
go install github.com/kyleconroy/sqlc/cmd/sqlc@latest
```
Then run `sqlc version` to make sure it's installed correctly.

### 2. Install Goose
Like SQLC, Goose is just a command line tool. I also recommend [installing](https://github.com/pressly/goose#install) it using `go install`:
```bash
go install github.com/pressly/goose/v3/cmd/goose@latest
```
Run `goose -version` to make sure it's installed correctly.

### 3. Create the `users` migration
I recommend creating an `sql` directory in the root of your project, and in there creating a `schema` directory.

A "migration" is a SQL file that describes a change to your database schema. For now, we need our first migration to create a `users` table. The simplest format for these files is:
```
number_name.sql
```
For example, I created a file in `sql/schema` called `001_users.sql` with the following contents:
```sql
-- +goose Up
CREATE TABLE ...

-- +goose Down
DROP TABLE users;
```
The `-- +goose Up` and `-- +goose Down` comments tell Goose how to run the migration. An "up" migration moves your database from its old state to a new state. A "down" migration moves your database from its new state back to its old state.

By running all of the "up" migrations on a blank database, you should end up with a database in a ready-to-use state. "Down" migrations are only used when you need to roll back a migration, or if you need to reset a local testing database to a known state.

### 4. Run the migration
`cd` into the `sql/schema` directory and run:
```bash
goose postgres CONN up
```
Where `CONN` is the connection string for your database. Ex :
```
protocol://username:password@host:port/database
```
Run your migration! Make sure it works by using PGAdmin to find your newly created `users` table.

### 5. Save your connection string as an environment variable
Add your connection string to your `.env` file :
```
protocol://username:password@host:port/database?sslmode=disable
```
Your application code needs to know to not try to use SSL locally.

### 6. Configure [SQLC](https://docs.sqlc.dev/en/latest/tutorials/getting-started-postgresql.html)
You'll always run the `sqlc` command from the root of your project. Create a file called `sqlc.yaml` in the root of your project. Ex :
```yaml
version: "2"
sql:
  - schema: "sql/schema"
    queries: "sql/queries"
    engine: "postgresql"
    gen:
      go:
        out: "internal/database"
```
We're telling SQLC to look in the `sql/schema` directory for our schema structure (which is the same set of files that Goose uses, but sqlc automatically ignores "down" migrations), and in the `sql/queries` directory for queries. We're also telling it to generate Go code in the `internal/database` directory.

### 7. Write a query to create a user
Inside the `sql/queries` directory, create a file called `users.sql`. Here is mine:
```sql
-- name: CreateUser :one
INSERT INTO users (id, created_at, updated_at, name)
VALUES ($1, $2, $3, $4)
RETURNING *;
```
`$1`, `$2`, `$3`, and `$4` are parameters that we'll be able to pass into the query in our Go code. The `:one` at the end of the query name tells SQLC that we expect to get back a single row (the created user).

Keep the [SQLC docs](https://docs.sqlc.dev/en/latest/tutorials/getting-started-postgresql.html) handy, you'll probably need to refer to them again later.

### 8. Generate the Go code
Run `sqlc generate` from the root of your project. It should create a new package of go code in `internal/database`.

### 9. Open a connection to the database, and store it in a config struct
It's common to use a "config" struct to store shared data that HTTP handlers need access to. Ex :
```go
type apiConfig struct {
	DB *database.Queries
}
```
At the top of `main()` load in your database URL from your `.env` file, and then [.Open()](https://pkg.go.dev/database/sql#Open) a connection to your database :
```go
db, err := sql.Open("postgres", dbURL)
```
Use your generated `database` package to create a new `*database.Queries`, and store it in your config struct :
```go
dbQueries := database.New(db)
```

### 10. Create an HTTP handler to create a user
Endpoint: `POST /v1/users`
Example body :
```json
{
  "name": "Lane"
}
```
Example response :
```json
{
    "id": "3f8805e3-634c-49dd-a347-ab36479f3f83",
    "created_at": "2021-09-01T00:00:00Z",
    "updated_at": "2021-09-01T00:00:00Z",
    "name": "Lane"
}
```
Use Google's [UUID](https://pkg.go.dev/github.com/google/uuid) package to generate a new [UUID](https://blog.boot.dev/clean-code/what-are-uuids-and-should-you-use-them/) for the user's ID. Both `created_at` and `updated_at` should be set to the current time. If we ever need to update a user, we'll update the `updated_at` field.

Follow a convention where *every table* in my database has:
* An `id` field that is a UUID (if you're curious why, [read this](https://blog.boot.dev/clean-code/what-are-uuids-and-should-you-use-them/))
* A `created_at` field that indicates when the row was created
* An `updated_at` field that indicates when the row was last updated

### 11. Add "api key" column to the users table
We'll generate valid API keys (256-bit hex values) using SQL. Ex :
```sql
encode(sha256(random()::text::bytea), 'hex')
```
New endpoint allows users to get their own user information by parsing the header and using new query to get the user data :

Endpoint: `GET /v1/users`

Request headers: `Authorization: ApiKey <key>`

Example response body:
```json
{
    "id": "3f8805e3-634c-49dd-a347-ab36479f3f83",
    "created_at": "2021-09-01T00:00:00Z",
    "updated_at": "2021-09-01T00:00:00Z",
    "name": "Lane",
    "api_key": "cca9688383ceaa25bd605575ac9700da94422aa397ef87e765c8df4438bc9942"
}
```

## Create a Feed
An RSS feed is just a URL that points to some XML. Users will be able to add feeds to our database so that our server (in a future step) can go download all of the posts in the feed (like blog posts or podcast episodes).

### 1. Create a feeds table
Like any table in our DB, we'll need the standard `id`, `created_at`, and `updated_at` fields. We'll also need a few more:
* `name`: The name of the feed (like "The Changelog, or "The Boot.dev Blog")
* `url`: The URL of the feed
* `user_id`: The ID of the user who added this feed

I'd recommend making the `url` field unique so that in the future we aren't downloading duplicate posts. I'd also recommend using [ON DELETE CASCADE](https://stackoverflow.com/a/14141354) on the `user_id` foreign key so that if a user is deleted, all of their feeds are automatically deleted as well.

Write the appropriate migrations and run them.

### 2. Create some authentication middleware

Most of the endpoints going forward will require a user to be logged in. Let's DRY up our code by creating some middleware that will check for a valid API key.

Currently, Chi router handles stateful middleware using [context](https://pkg.go.dev/context) (middleware that passes data down to the next handler). Preference is to create custom handlers that accept extra values. You can add middleware however you like. Examples :
#### A custom type for handlers that require authentication
```go
type authedHandler func(http.ResponseWriter, *http.Request, database.User)
```
#### Middleware that authenticates a request, gets the user and calls the next authed handler
```go
func (cfg *apiConfig) middlewareAuth(handler authedHandler) http.HandlerFunc {
    ///
}
```
#### Using the middleware
```go
v1Router.Get("/users", apiCfg.middlewareAuth(apiCfg.handlerUsersGet))
```

## Feed Follows

Aside from just adding new feeds to the database, users can specify *which* feeds they want to follow. This will be important later when we want to show users a list of posts from the feeds they follow.

A feed follow is just a link between a user and a feed. It's a [many-to-many](https://en.wikipedia.org/wiki/Many-to-many_(data_model)) relationship, so a user can follow many feeds, and a feed can be followed by many users.

Creating a feed follow indicates that a user is now following a feed. Deleting it is the same as "unfollowing" a feed.

It's important to understand that the `ID` of a feed follow is not the same as the `ID` of the feed itself. Each user/feed pair will have a unique feed follow id.

### Create a feed follow
Endpoint: `POST /v1/feed_follows`

*Requires authentication*

Example request body:
```json
{
  "feed_id": "4a82b372-b0e2-45e3-956a-b9b83358f86b"
}
```
Example response body:
```json
{
  "id": "c834c69e-ee26-4c63-a677-a977432f9cfa",
  "feed_id": "4a82b372-b0e2-45e3-956a-b9b83358f86b",
  "user_id": "0e4fecc6-1354-47b8-8336-2077b307b20e",
  "created_at": "2017-01-01T00:00:00Z",
  "updated_at": "2017-01-01T00:00:00Z"
}
```

### Delete a feed follow
Endpoint: `DELETE /v1/feed_follows/{feedFollowID}`

### Get all feed follows for a user
Endpoint: `GET /v1/feed_follows`

## Scraper

### Add a `last_fetched_at` column to the `feeds` table
We need to keep track of when we last fetched the posts from a feed. This should be a nullable timestamp.

The `sql.NullTime` type is useful for nullable timestamps on the database side, but it's not great for marshaling into JSON. It results in a weird nested object. I'd recommend converting it to a `*time.Time` before returning it across the HTTP response.

I map all of my database structs to a different struct that has the intended JSON structure. This is a good way to keep your database and HTTP APIs separate.
For example: `func databaseFeedToFeed(feed database.Feed) Feed`

### Add `GetNextFeedsToFetch()` query to the database
It should return the next `n` feeds that need to be fetched, ordered by `last_fetched_at`, but with `NULL` values first. We obviously want to fetch the feeds that have never been fetched before or the ones that were fetched the longest time ago.

### Add a `MarkFeedFetched()` query to the database
It should update a feed and set its `last_fetched_at` to the current time. Don't forget to also update the `updated_at` field because we've updated the record.

### Write a function that can fetch data from a feed
This function should accept the URL of a live RSS feed, and return the parsed data in a Go struct. Ex:
* `https://blog.boot.dev/index.xml`
* `https://wagslane.com/feed.xml`

You can parse the returned XML with the [encoding/xml](https://pkg.go.dev/encoding/xml) package, it works *very* similarly to `encoding/json`. Define the structure of an RSS feed as a Go struct, then unmarshal the XML into that struct.

### Write a worker that fetches feeds continuously
This function should, on an interval (say every 60 seconds or so):
* Get the next `n` feeds to fetch from the database (you can configure `n`, I used `10`)
* Fetch and process all the feeds *at the same time* (you can use [sync.WaitGroup](https://pkg.go.dev/sync#WaitGroup) for this)

For now, "process" the feed by simply printing out the titles of each post

### Call your worker from `main.go`
Be sure to start the worker in its own goroutine, so that it runs in the background and processes feeds even as it simultaneously handles new HTTP requests.

## Posts

### Add a `posts` table to the database
A post is a single entry from a feed. It should have:
* `id` - a unique identifier for the post
* `created_at` - the time the record was created
* `updated_at` - the time the record was last updated
* `title` - the title of the post
* `url` - the URL of the post *this should be unique*
* `description` - the description of the post
* `published_at` - the time the post was published
* `feed_id` - the ID of the feed that the post came from

Some of these fields can probably be null, others you might want to be more strict about - it's up to you.

### Add a "create post" SQL query to the database
This should insert a new post into the database.

### Add a "get posts by user" SQL query to the database
Order the results so that the most recent posts are first. Make the number of posts returned configurable.

### Update your scraper to save posts
Instead of just printing out the titles of the posts, save them to the database! If you encounter an error where the post with that URL already exists, just ignore it. That will happen a lot. If it's a different error, you should probably log it.

Make sure that you're parsing the "published at" time properly from the feeds. Sometimes they might be in a different format than you expect, so you might need to handle that.

### Add a "get posts by user" HTTP endpoint
Endpoint: `GET /v1/posts`

*This is an authenticated endpoint*

This endpoint should return a list of posts for the authenticated user. It should accept a `limit` query parameter that limits the number of posts returned. The default if the parameter is not provided can be whatever you think is reasonable.

### Start scraping some feeds!
Test your scraper to make sure it's working! Go find some of your favorite websites and add their RSS feeds to your database. Then start your scraper and watch it go to work.

## Ideas for extending the project
* Support [pagination](https://nordicapis.com/everything-you-need-to-know-about-api-pagination/) of the endpoints that can return many items
* Support different options for sorting and filtering posts using query parameters
* Classify different types of feeds and posts (e.g. blog, podcast, video, etc.)
* Add a CLI client that uses the API to fetch and display posts, maybe it even allows you to read them in your terminal
* Scrape lists of feeds themselves from a third-party site that aggregates feed URLs
* Add support for other types of feeds (e.g. Atom, JSON, etc.)
* Add integration tests that use the API to create, read, update, and delete feeds and posts
* Add bookmarking or "liking" to posts
* Create a simple web UI that uses your backend API
