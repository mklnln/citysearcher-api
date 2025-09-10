# Citysearcher API

## What is this?

Citysearcher is a little web app that pings a database with a string of letters and a latitude/longitude position. It returns suggestions of cities and scores these suggestions by how close they are in terms of a name match (e.g. "Mon" is close to "Montreal" but "Montr" is even better) and the latitude/longitude match.

### Why?

Well, if a user is wanting to get information about a place, we can make a reasonable guess that they're more interested in results that are nearer to them. If we have their position, we can use that information to provide a better user experience by guessing they want to see something nearby.

### Example

See the live site at https://citysearcher-api.onrender.com/suggestions?q=Montreal

The database may take a good 60-90 seconds to respond upon cold start since it's built using the free tiers of hosting services. **_If you get a 502 Bad Gateway error, give it another minute and try again_**. The server is hosted on [Render](https://render.com/) and the database is hosted on [Aiven](https://aiven.io/).

Adding latitude and longitude to the URL will enable the scoring algorithm for the results.

-   e.g. https://citysearcher-api.onrender.com/suggestions?q=Lon&latitude=43.70011&longitude=-79.4163

## Tech used

This TypeScript Express server (hosted on Render) uses Prisma to send SQL queries. A Postgres database responds with JSON.

---

## To get started, run Docker locally

You'll need Node v20.4.0, TypeScript and Docker (verify you have working `npm`, `tsc`, and `docker` commands).

In repo directory:

-   `nvm use` and `npm install`
-   Ensure you have the `.env` file in root directory
-   `npm run build` starts pulling the image and drawing up a containerized DB **_and_** server
-   Upon seeing `Listening on port 4000`, open a new terminal for `npm run db-init`
-   After a moment, the server should be listening and the seed command should have executed
-   Test at http://localhost:4000/suggestions?q=Mont

Note that subsequent docker compose up and down commands (from an `npm run xyz`) might create errors from the Postgres terminal about duplicate keys. At that point, just use `npm run server`.

This local repo has been tested on 3 machines, Pop!\_OS 22.04 and two different Windows 10 installations. All worked without issue.

An **M1 Mac** came across the following error and was unable to seed the DB: https://github.com/prisma/prisma/issues/18207. I don't actually own the M1 Mac and so can't offer much help on troubleshooting it.

---

## Testing

To call the entire test suite, `npm run test` while the `Listening on port 4000` terminal is active.

Tests were added to:

-   Verify SQL injection doesn't work
-   Ensure latitude/longitude values are between -90 and 90
-   Show that coordinates provide a more confident autocomplete suggestion
-   Confirm that a 429 Too Many Requests error code can occur

---

# Architecture Decisions

## Why Prisma?

Prisma makes it really easy to both migrate schemas and seed databases. Prisma can integrate with various SQL databases, can be used as a query builder, and helps enforce client side type safety.

$queryRaw is a way to manually write SQL queries for when the query builder isn't enough. It allows for conditional rendering of SQL commands as well as template string interpolation, see /src/findCities.ts lines 33 and 49.

"**... but can't that lead to SQL injection??**"

Nope!

```
const query = '; DROP TABLES "City";--'

const result = await prisma.$queryRaw`${query}`
```

Prisma plans for that by making any interpolated variables (i.e. our `const query`) **only** be processed as data and never as SQL keywords. This means that `const result` will create a syntax error. It reads the SQL query as just a string, no keywords. Without any keywords, nothing can be deleted or modified. The database is safe.

### Key Ideas

-   The Schema.prisma file contains the definition of the database schema (in Prisma called a model) which will be used to populate the database with the correct table fields and types, resulting in a very easy TypeScript integration
-   /Prisma/seed.ts enjoys a very simple prisma.city.create query builder that seeds the database from the parsed TSV file
-   SQL injection is taken care of
