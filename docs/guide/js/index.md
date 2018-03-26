# Reactive Postgres Client

The Reactive Postgres Client is a client for Postgres with a straightforward API focusing on
scalability and low overhead.

The client is reactive and non blocking, allowing to handle many database connections with a single thread.

* Event driven
* Lightweight
* Built-in connection pooling
* Prepared queries caching
* Publish / subscribe using Postgres `NOTIFY/LISTEN`
* Batch and cursor support
* Row streaming
* Command pipeling
* RxJava 1 and RxJava 2 support
* Direct memory to object without unnecessary copies
* Java 8 Date and Time support
* SSL/TLS support
* HTTP/1.x CONNECT, SOCKS4a or SOCKS5 proxy support

## Usage

To use the Reactive Postgres Client add the following dependency to the _dependencies_ section of your build descriptor:

* Maven (in your `pom.xml`):

[source,xml,subs#"+attributes"]
```
<dependency>
 <groupId>io.reactiverse</groupId>
 <artifactId>reactive-pg-client</artifactId>
 <version>0.7.0</version>
</dependency>
```

* Gradle (in your `build.gradle` file):

```groovy
dependencies {
 compile 'io.reactiverse:reactive-pg-client:0.7.0'
}
```

## Getting started

Here is the simplest way to connect, query and disconnect

```js
var PgClient = require("reactive-pg-client-js/pg_client");

// Pool options
var options = {
  "port" : 5432,
  "host" : "the-host",
  "database" : "the-db",
  "username" : "user",
  "password" : "secret",
  "maxSize" : 5
};

// Create the client pool
var client = PgClient.pool(options);

// A simple query
client.query("SELECT * FROM users WHERE id='julien'", function (ar, ar_err) {
  if (ar_err == null) {
    var result = ar;
    console.log("Got " + result.size() + " results ");
  } else {
    console.log("Failure: " + ar_err.getMessage());
  }

  // Now close the pool
  client.close();
});

```

## Connecting to Postgres

Most of the time you will use a pool to connect to Postgres:

```js
var PgClient = require("reactive-pg-client-js/pg_client");

// Pool options
var options = {
  "port" : 5432,
  "host" : "the-host",
  "database" : "the-db",
  "username" : "user",
  "password" : "secret",
  "maxSize" : 5
};

// Create the pooled client
var client = PgClient.pool(options);

```

The pooled client uses a connection pool and any operation will borrow a connection from the pool
to execute the operation and release it to the pool.

If you are running with Vert.x you can pass it your Vertx instance:

```js
var PgClient = require("reactive-pg-client-js/pg_client");

// Pool options
var options = {
  "port" : 5432,
  "host" : "the-host",
  "database" : "the-db",
  "username" : "user",
  "password" : "secret",
  "maxSize" : 5
};

// Create the pooled client
var client = PgClient.pool(vertx, options);

```

You need to release the pool when you don't need it anymore:

```js

// Close the pool and all the associated resources
pool.close();

```

When you need to execute several operations on the same connection, you need to use a client
[`connection`](../../jsdoc/module-reactive-pg-client-js_pg_connection-PgConnection.html).

You can easily get one from the pool:

```js
var PgClient = require("reactive-pg-client-js/pg_client");

// Pool options
var options = {
  "port" : 5432,
  "host" : "the-host",
  "database" : "the-db",
  "username" : "user",
  "password" : "secret",
  "maxSize" : 5
};

// Create the pooled client
var client = PgClient.pool(vertx, options);

// Get a connection from the pool
client.getConnection(function (ar1, ar1_err) {

  if (ar1_err == null) {

    console.log("Connected");

    // Obtain our connection
    var conn = ar1;

    // All operations execute on the same connection
    conn.query("SELECT * FROM users WHERE id='julien'", function (ar2, ar2_err) {
      if (ar2_err == null) {
        conn.query("SELECT * FROM users WHERE id='emad'", function (ar3, ar3_err) {
          // Release the connection to the pool
          conn.close();
        });
      } else {
        // Release the connection to the pool
        conn.close();
      }
    });
  } else {
    console.log("Could not connect: " + ar1_err.getMessage());
  }
});

```

Once you are done with the connection you must close it to release it to the pool, so it can be reused.

## Running queries

When you don't need a transaction or run single queries, you can run queries directly on the pool; the pool
will use one of its connection to run the query and return the result to you.

Here is how to run simple queries:

```js
client.query("SELECT * FROM users WHERE id='julien'", function (ar, ar_err) {
  if (ar_err == null) {
    var result = ar;
    console.log("Got " + result.size() + " results ");
  } else {
    console.log("Failure: " + ar_err.getMessage());
  }
});

```

You can do the same with prepared queries.

The SQL string can refer to parameters by position, using `$1`, `$2`, etc…​

```js
var Tuple = require("reactive-pg-client-js/tuple");
client.preparedQuery("SELECT * FROM users WHERE id=$1", Tuple.of("julien"), function (ar, ar_err) {
  if (ar_err == null) {
    var result = ar;
    console.log("Got " + result.size() + " results ");
  } else {
    console.log("Failure: " + ar_err.getMessage());
  }
});

```

Query methods provides an asynchronous [`PgResult`](../../jsdoc/module-reactive-pg-client-js_pg_result-PgResult.html) instance that works for _SELECT_ queries

```js
client.preparedQuery("SELECT first_name, last_name FROM users", function (ar, ar_err) {
  if (ar_err == null) {
    var result = ar;
    Array.prototype.forEach.call(result, function(row) {
      console.log("User " + row.getString(0) + " " + row.getString(1));
    });
  } else {
    console.log("Failure: " + ar_err.getMessage());
  }
});

```

or _UPDATE_/_INSERT_ queries:

```js
var Tuple = require("reactive-pg-client-js/tuple");
client.preparedQuery("INSERT INTO users (first_name, last_name) VALUES ($1, $2)", Tuple.of("Julien", "Viet"), function (ar, ar_err) {
  if (ar_err == null) {
    var result = ar;
    console.log(result.updatedCount());
  } else {
    console.log("Failure: " + ar_err.getMessage());
  }
});

```

The [`Row`](../../jsdoc/module-reactive-pg-client-js_row-Row.html) gives you access to your data by index

```js
console.log("User " + row.getString(0) + " " + row.getString(1));

```

or by name

```js
console.log("User " + row.getString("first_name") + " " + row.getString("last_name"));

```

You can access a wide variety of of types

```js

var firstName = row.getString("first_name");
var male = row.getBoolean("male");
var age = row.getInteger("age");

// ...


```

You can execute prepared batch

```js
var Tuple = require("reactive-pg-client-js/tuple");

// Add commands to the batch
var batch = [];
batch.push(Tuple.of("julien", "Julien Viet"));
batch.push(Tuple.of("emad", "Emad Alblueshi"));

// Execute the prepared batch
client.preparedBatch("INSERT INTO USERS (id, name) VALUES ($1, $2)", batch, function (res, res_err) {
  if (res_err == null) {

    // Process results
    var results = res;
  } else {
    console.log("Batch failed " + res_err);
  }
});

```

You can cache prepared queries:

```js
var PgClient = require("reactive-pg-client-js/pg_client");

// Enable prepare statements
options.cachePreparedStatements = true;

var client = PgClient.pool(vertx, options);

```

## Using connections

When you need to execute sequential queries (without a transaction), you can create a new connection
or borrow one from the pool:

```js
Code not translatable
```

Prepared queries can be created:

```js
var Tuple = require("reactive-pg-client-js/tuple");
connection.prepare("SELECT * FROM users WHERE first_name LIKE $1", function (ar1, ar1_err) {
  if (ar1_err == null) {
    var pq = ar1;
    pq.execute(Tuple.of("julien"), function (ar2, ar2_err) {
      if (ar2_err == null) {
        // All rows
        var result = ar2;
      }
    });
  }
});

```

NOTE: prepared query caching depends on the [`cachePreparedStatements`](../dataobjects.html#PgConnectOptions#setCachePreparedStatements) and
does not depend on whether you are creating prepared queries or use [`direct prepared queries`](../../jsdoc/module-reactive-pg-client-js_pg_client-PgClient.html#preparedQuery)

By default prepared query executions fetch all results, you can use a [`PgCursor`](../../jsdoc/module-reactive-pg-client-js_pg_cursor-PgCursor.html) to control the amount of rows you want to read:

```js
var Tuple = require("reactive-pg-client-js/tuple");
connection.prepare("SELECT * FROM users WHERE first_name LIKE $1", function (ar1, ar1_err) {
  if (ar1_err == null) {
    var pq = ar1;

    // Create a cursor
    var cursor = pq.cursor(Tuple.of("julien"));

    // Read 50 rows
    cursor.read(50, function (ar2, ar2_err) {
      if (ar2_err == null) {
        var result = ar2;

        // Check for more ?
        if (cursor.hasMore()) {

          // Read the next 50
          cursor.read(50, function (ar3, ar3_err) {
            // More results, and so on...
          });
        } else {
          // No more results
        }
      }
    });
  }
});

```

Cursors shall be closed when they are released prematurely:

```js
var Tuple = require("reactive-pg-client-js/tuple");
connection.prepare("SELECT * FROM users WHERE first_name LIKE $1", function (ar1, ar1_err) {
  if (ar1_err == null) {
    var pq = ar1;
    var cursor = pq.cursor(Tuple.of("julien"));
    cursor.read(50, function (ar2, ar2_err) {
      if (ar2_err == null) {
        // Close the cursor
        cursor.close();
      }
    });
  }
});

```

A stream API is also available for cursors, which can be more convenient, specially with the Rxified version.

```js
var Tuple = require("reactive-pg-client-js/tuple");
connection.prepare("SELECT * FROM users WHERE first_name LIKE $1", function (ar1, ar1_err) {
  if (ar1_err == null) {
    var pq = ar1;

    // Fetch 50 rows at a time
    var stream = pq.createStream(50, Tuple.of("julien"));

    // Use the stream
    stream.exceptionHandler(function (err) {
      console.log("Error: " + err.getMessage());
    });
    stream.endHandler(function (v) {
      console.log("End of stream");
    });
    stream.handler(function (row) {
      console.log("User: " + row.getString("last_name"));
    });
  }
});

```

The stream read the rows by batch of `50` and stream them, when the rows have been passed to the handler,
a new batch of `50` is read and so on.

The stream can be resumed or paused, the loaded rows will remain in memory until they are delivered and the cursor
will stop iterating.

[`PgPreparedQuery`](../../jsdoc/module-reactive-pg-client-js_pg_prepared_query-PgPreparedQuery.html)can perform efficient batching:

```js
var Tuple = require("reactive-pg-client-js/tuple");
connection.prepare("INSERT INTO USERS (id, name) VALUES ($1, $2)", function (ar1, ar1_err) {
  if (ar1_err == null) {
    var prepared = ar1;

    // Create a query : bind parameters
    var batch = [];

    // Add commands to the createBatch
    batch.push(Tuple.of("julien", "Julien Viet"));
    batch.push(Tuple.of("emad", "Emad Alblueshi"));

    prepared.batch(batch, function (res, res_err) {
      if (res_err == null) {

        // Process results
        var results = res;
      } else {
        console.log("Batch failed " + res_err);
      }
    });
  }
});

```

## Using transactions

You can execute transaction using SQL `BEGIN`/`COMMIT`/`ROLLBACK`, if you do so you must use
a [`PgConnection`](../../jsdoc/module-reactive-pg-client-js_pg_connection-PgConnection.html) and manage it yourself.

Or you can use the transaction API of [`PgConnection`](../../jsdoc/module-reactive-pg-client-js_pg_connection-PgConnection.html):

```js
Code not translatable
```

When Postgres reports the current transaction is failed (e.g the infamous _current transaction is aborted, commands ignored until
end of transaction block_), the transaction is rollbacked and the [`abortHandler`](../../jsdoc/module-reactive-pg-client-js_pg_transaction-PgTransaction.html#abortHandler)
is called:

```js
pool.getConnection(function (res, res_err) {
  if (res_err == null) {

    // Transaction must use a connection
    var conn = res;

    // Begin the transaction
    var tx = conn.begin().abortHandler(function (v) {
      console.log("Transaction failed => rollbacked");
    });

    conn.query("INSERT INTO Users (first_name,last_name) VALUES ('Julien','Viet')", function (ar, ar_err) {
      // Works fine of course
    });
    conn.query("INSERT INTO Users (first_name,last_name) VALUES ('Julien','Viet')", function (ar, ar_err) {
      // Fails and triggers transaction aborts
    });

    // Attempt to commit the transaction
    tx.commit(function (ar, ar_err) {
      // But transaction abortion fails it
    });
  }
});

```

## Postgres type mapping

### Handling JSON

The [`Json`](../../jsdoc/module-reactive-pg-client-js_json-Json.html) Java type is used to represent the Postgres `JSON` and `JSONB` type.

The main reason of this type is handling `null` JSON values.

```js
var Json = require("reactive-pg-client-js/json");
var Tuple = require("reactive-pg-client-js/tuple");

// Create a tuple
var tuple = Tuple.of(Json.create(null), Json.create({
  "foo" : "bar"
}), Json.create(3));

//
tuple = Tuple.tuple().addJson(Json.create(null)).addJson(Json.create({
  "foo" : "bar"
})).addJson(Json.create(3));

// JSON object (and arrays) can also be added directly
tuple = Tuple.tuple().addJson(Json.create(null)).addJsonObject({
  "foo" : "bar"
}).addJson(Json.create(3));

// Retrieving json
var value = tuple.getJson(0).value();

//
value = tuple.getJson(1).value();
value = tuple.getJsonObject(1);

//
value = tuple.getJson(3).value();

```

### Handling NUMERIC

The `Numeric` Java type is used to represent the Postgres `NUMERIC` type.

```js
var numeric = row.getNumeric("value");
if (numeric.isNaN()) {
  // Handle NaN
} else {
  var value = numeric.bigDecimalValue();
}

```

## Pub/sub

Postgres supports pub/sub communication channels.

You can set a [`notificationHandler`](../../jsdoc/module-reactive-pg-client-js_pg_connection-PgConnection.html#notificationHandler) to receive
Postgres notifications:

```js

connection.notificationHandler(function (notification) {
  console.log("Received " + notification.payload + " on channel " + notification.channel);
});

connection.query("LISTEN some-channel", function (ar, ar_err) {
  console.log("Subscribed to channel");
});

```

The [`PgSubscriber`](../../jsdoc/module-reactive-pg-client-js_pg_subscriber-PgSubscriber.html) is a channel manager managing a single connection that
provides per channel subscription:

```js
var PgSubscriber = require("reactive-pg-client-js/pg_subscriber");

var subscriber = PgSubscriber.subscriber(vertx, {
  "port" : 5432,
  "host" : "the-host",
  "database" : "the-db",
  "username" : "user",
  "password" : "secret"
});

// You can set the channel before connect
subscriber.channel("channel1").handler(function (payload) {
  console.log("Received " + payload);
});

subscriber.connect(function (ar, ar_err) {
  if (ar_err == null) {

    // Or you can set the channel after connect
    subscriber.channel("channel2").handler(function (payload) {
      console.log("Received " + payload);
    });
  }
});

```

You can provide a reconnect policy as a function that takes the number of `retries` as argument and returns an `amountOfTime`
value:

* when `amountOfTime < 0`: the subscriber is closed and there is no retry
* when `amountOfTime ## 0`: the subscriber retries to connect immediately
* when `amountOfTime > 0`: the subscriber retries after `amountOfTime` milliseconds

```js
var PgSubscriber = require("reactive-pg-client-js/pg_subscriber");

var subscriber = PgSubscriber.subscriber(vertx, {
  "port" : 5432,
  "host" : "the-host",
  "database" : "the-db",
  "username" : "user",
  "password" : "secret"
});

// Reconnect at most 10 times after 100 ms each
subscriber.reconnectPolicy(function (retries) {
  if (retries < 10) {
    return 100
  } else {
    return -1
  }
});

```

The default policy is to not reconnect.

## Using SSL/TLS

To configure the client to use SSL connection, you can configure the [`PgConnectOptions`](../dataobjects.html#PgConnectOptions)
like a Vert.x `NetClient`}.

```js
var PgClient = require("reactive-pg-client-js/pg_client");

var options = {
  "port" : 5432,
  "host" : "the-host",
  "database" : "the-db",
  "username" : "user",
  "password" : "secret",
  "ssl" : true,
  "pemTrustOptions" : {
    "certPaths" : [
      "/path/to/cert.pem"
    ]
  }
};

PgClient.connect(vertx, options, function (res, res_err) {
  if (res_err == null) {
    // Connected with SSL
  } else {
    console.log("Could not connect " + res_err);
  }
});

```

More information can be found in the [Vert.x documentation](http://vertx.io/docs/vertx-core/java/#ssl).

## Using a proxy

You can also configure the client to use an HTTP/1.x CONNECT, SOCKS4a or SOCKS5 proxy.

More information can be found in the http://vertx.io/docs/vertx-core/java/#_using_a_proxy_for_client_connections[Vert.x documentation].