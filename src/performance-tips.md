---
title: "Performance Tips"
last_updated: 2015-11-06
---

## Repeated Queries

In general, you will get better performance out of your database if you
group together similar queries.

For example, if your application is making these queries:

```sql
SELECT * FROM "users" WHERE "id" = 12;
SELECT * FROM "users" WHERE "id" = 15;
SELECT * FROM "users" WHERE "id" = 27;
```

It would be more efficient to make a single SQL query:

```sql
SELECT * FROM "users" WHERE "id" IN (12, 15, 27);
```

Skylight automatically normalizes all three of the above repeated
queries into the following query, allowing us to detect the repetition:

```sql
SELECT * FROM "users" WHERE "id" = ?;
```

### Possible Cause: N+1 Query

A very common cause of repeated queries in Rails applications is known
as "N+1 Queries". This happens because you make a request for a single
row in one table, and then make an additional request per element in a
`has_many` relationship.

Here's an example offender ([from the Rails guides][1]):

[1]: http://guides.rubyonrails.org/active_record_querying.html#eager-loading-associations

```rb
def show
  client = Client.limit(10)
  @postcodes = client.map { |c| c.address.postcode }
end
```

The mistake here is that you're making a single query for 10 clients,
but then one query for each client to get its address, something like
this:

```sql
SELECT * from "clients" LIMIT 10;
SELECT * from "addresses" WHERE "id" = 7;
SELECT * from "addresses" WHERE "id" = 8;
SELECT * from "addresses" WHERE "id" = 10;
SELECT * from "addresses" WHERE "id" = 12;
SELECT * from "addresses" WHERE "id" = 13;
SELECT * from "addresses" WHERE "id" = 15;
SELECT * from "addresses" WHERE "id" = 16;
SELECT * from "addresses" WHERE "id" = 17;
SELECT * from "addresses" WHERE "id" = 21;
```

Skylight normalizes these queries into the following, and if it happens
across multiple requests, reports a repeated query:

```sql
SELECT * from "clients" LIMIT ?;
SELECT * from "addresses" WHERE "id" = ?; // x 10
```

#### The Solution

The solution to this problem is ["eager loading"][2], which means
specifying ahead of time which associations you will need.

[2]: http://api.rubyonrails.org/classes/ActiveRecord/QueryMethods.html#method-i-includes

```rb
def show
  client = Client.includes(:address).limit(10)
  @postcodes = client.map { |c| c.address.postcode }
end
```

Now, Rails will generate the following SQL for you ahead of time, before
you get to the `map`:

```rb
SELECT * from "clients" LIMIT 10;
SELECT * from "addresses" WHERE "client_id" IN (7, 8, 10, 12, 13, 15, 16, 17, 21);
```

### Other Possiblities

Skylight will report any kind of repeated query that includes more than
4 repetitions, happens on a regular basis, and consumes a large part of
the total request.

This includes those involving `INSERT` statements. We tuned the
heuristic for reporting the problems so that virtually all reported
problems will benefit from grouping.

Not all repeated queries can be resolved using the "eager loading"
technique described above, but because Skylight only reports repeated
queries that consume a large part of the total request, your app will
likely benefit from fixing it.
