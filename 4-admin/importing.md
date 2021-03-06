---
layout: documentation
title: Importing your data
docs_active: importing
permalink: docs/importing/
---

<img alt="Importing Data Illustration" class="api_command_illustration"
    src="/assets/images/docs/api_illustrations/importing_data.png" />

The `rethinkdb` utility includes an `import` command to load existing data into RethinkDB databases. It can read JSON files, organized in one of two formats described below, or comma-separated value (CSV) files (including ones with other delimiters such as tab characters).

When the option is available, you should choose the JSON file format. If you're exporting from a SQL database this might not be possible, but you might be able to write a separate script to transform CSV output into JSON, or use the `mysql2json` script available as part of [mysql2xxxx][m2x].

[m2x]: https://github.com/seamusabshere/mysql2xxxx

The full syntax for the `import` command is as follows:

    rethinkdb import -f FILE --table DB.TABLE [-c HOST:PORT] [-a AUTH_KEY]
      [--force] [--clients NUM] [--format (csv | json)] [--pkey PRIMARY_KEY]
      [--delimiter CHARACTER] [--custom-header FIELD,FIELD... [--no-header]]

To import the file `users.json` into the table `test.users`, you would use:

    rethinkdb import -f users.json --table test.users

If it were a CSV file, you would use:

    rethinkdb import -f users.csv --format csv --table test.users

By default, the import command will connect to `localhost` port `28015`. You can use the `-c` option to specify a server and client port to connect to. (Note this is the driver port clients connect to, not the cluster port.)

    rethinkdb import -f crew.json --table discovery.crew -c hal:2001

If the cluster requires authorization, you can specify the authorization key with the `-a` option.

    rethinkdb import -f crew.json --table discovery.crew -c hal:2001 -a daisy

A primary key other than `id` can be specified with `--pkey`:

    rethinkdb import -f heroes.json --table marvel.heroes --pkey name

JSON files are preferred to CSV files, as JSON can represent RethinkDB documents fully. If you're importing from a CSV file, you should include a header row with the field names, or use the `--no-header` option with the `--custom-header` option to specify the names.

    rethinkdb import -f users.csv --format csv --table test.users --no-header \
        --custom-header id,username,email,password

The CSV delimiter defaults to the comma, but this can be overridden with the `--delimiter` option. Use `--delimiter '\t'` for a tab-delimited file.

Values in CSV imports will always be imported as strings. If you want to convert those fields after import to the `number` data type, run an `update` query that does the conversion. An example runnable in the Data Explorer:

```js
r.table('tablename').update(function(doc) {
    return doc.merge({
        field1: doc('field1').coerceTo('number'),
        field2: doc('field2').coerceTo('number')
    })
});
```

RethinkDB will accept two formats for JSON files:

* An array of JSON documents.

    ```js
    [ { field: "value" }, { field: "value"}, ... ]
    ```

* Whitespace-separated JSON rows.

    ```js
    { field: "value" }
    { field: "value" }
    ```

In both cases, each documents is a JSON object, bracketed with `{ }` characters. Only the first format is itself a valid JSON document, but RethinkDB will import documents properly either way.

There are more options than what we've covered here. Run `rethinkdb help import` for a full list of parameters and examples.

{% infobox alert %}

While `import` has the ability to import a directory full of files, those files are expected to be in the format and directory structure created by the `export` command.

{% endinfobox %}
