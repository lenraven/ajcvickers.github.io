---
layout: default
title: "Introducing EF Core 5 Features: CreateDbCommand: I'll see your string and raise you a command..."
date: 2020-01-15 16:01
day: 15th
month: January
year: 2020
author: ajcvickers
permalink: 2020/01/15/createdbcommand/
---

# EF Core 5.0
# CreateDbCommand: I'll see your string and raise you a command...

My <a href="/2020/01/12/toquerystring/">previous post</a> showed using <code>ToQueryString</code> to get generated SQL. This will commonly be copy-pasted, but it could also be executed directly by the application. For example:

```c#
var city = "London";
var query = context.Customers.Where(c => c.City == city);

var connection = context.Database.GetDbConnection();
using (var command = connection.CreateCommand())
{
    command.CommandText = query.ToQueryString();

    connection.Open();
    using (var results = command.ExecuteReader())
    {
        // Do something...
    }
    connection.Close();
}
```

This will work in simple cases, but the translation to a query string and back to a command loses some information. For example, if a transaction is being used then the code above would need to find that transaction and associate itself.

Instead, EF Core 5.0 introduces <code>CreateDbCommand</code> which creates and configures a <code>DbCommand</code> just as EF does to execute the query. For example:

``` c#
var city = "London";
var query = context.Customers.Where(c => c.City == city);

var connection = context.Database.GetDbConnection();
using (var command = query.CreateDbCommand())
{
    connection.Open();
    using (var results = command.ExecuteReader())
    {
        // Do something...
    }
    connection.Close();
}
```

Using this code the command is already configured with transactions, etc. Also, the parameters are configured on the command in exactly the way EF Core does to run the query. This makes <code>CreateDbCommand</code> the highest fidelity way of exactly getting the <code>DbCommand</code> what EF would use.

A few things to note:

<ul>
<li>This is not the actual <code>DbCommand</code> <em>instance</em> that EF will use to execute the query. If EF executes the query, then it will create a new <code>DbCommand</code> instance to do so.</li>
<li>The command must be disposed by application code, just as if it had been manually created.</li>
<li><code>CreateDbCommand</code> is only available for relational database providers. <code>ToQueryString</code>, on the other hand, also works with the Cosmos provider.</li>
</ul>

<h2>Caution!</h2>

Be careful what you do with generated commands. EF Core coordinates the setup of the command with the expected form and shape of the results. It may not always be what you are expecting! It also may change as EF internals continue to evolve.
