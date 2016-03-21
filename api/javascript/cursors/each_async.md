---
layout: api-command
language: JavaScript
permalink: api/javascript/each_async/
command: eachAsync
io:
    -   - cursor
        - undefined
related_commands:
    each: each/
---

# Command syntax #

{% apibody %}
sequence.eachAsync(function[, errorFunction]) &rarr; promise
{% endapibody %}

# Description #

Lazily iterate over a cursor, array or feed one element at a time. `eachAsync` always returns a promise that will be resolved once all rows are returned.

The first, required function passed to `eachAsync` takes one or two arguments, both functions: a callback to process each row as it is emitted, and an optional callback which will be executed when all row processing is completed.

```js
function(rowProcess[, final])
```

The `rowProcess` callback receives the row as its first argument; it may also take an optional second argument, which is a callback function to be executed when all rows have been processed.

```js
function(row[, rowFinished])
```

If you accept the `rowFinished` callback, it _must_ be called at the end of each row. If you call `rowFinished` with any value, iteration will stop, and the value will be wrapped in `error.message` for the error handler.

If you do _not_ use `rowFinished`, the `rowProcess` callback can end iteration early by returning any value _other_ than a Promise; that value will be wrapped in an error object and passed to `final` if it is provided. If it returns a Promise, the Promise will be resolved before iteration continues. (If the resolved Promise returns a value, iteration will be stopped and the value will be wrapped in an error object and passed to `final` if it is provided.)

If you provide a `final` callback, it will always be executed when row processing is completed (the end of the sequence is hit, iteration is stopped prematurely, or an error occurs). The `final` callback will receive an `error` object if an error is thrown or `rowProcess` returns any value (other than a Promise). If `final` returns a value, it will be passed on to any Promise chained after it (e.g., with a `.then()` method); if `final` returns a Promise, it will be resolved before iteration ends, and any value returned by that Promise will be passed on.

To summarize all of the above in code:

```js
// process each row asynchronously
cursor.eachAsync(function (row) {
    doSomethingWith(row);
});

// as above, but using rowFinished callback
cursor.eachAsync(function (row, rowFinished) {
    doSomethingWith(row);
    rowFinished();
});

// as above, but using final callback
cursor.eachAsync(function (row, rowFinished) {
    doSomethingWith(row);
    rowFinished();
}, function (final) {
    // the 'final' argument will only be defined when there is an error
    console.log('Final called with:', final);
});
```

__Example:__ Process all the elements in a stream, using `then` and `catch` for handling the end of the stream and any errors. Note that iteration may be stopped in the first callback (`rowProcess`) by returning any non-Promise value.

```js
cursor.eachAsync(function (row) {
    var ok = processRowData(row);
    if (!ok) {
        return 'Bad row: ' + row;
    } 
}).then(function () {
    console.log('done processing'); 
}).catch(function (error) {
    console.log('Error:', error.message);
});
```

__Example:__ As above, but using the `rowFinished` and `final` callbacks rather than the Promise returned from `eachAsync`.

```js
cursor.eachAsync(function (row, rowFinished) {
    var ok = processRowData(row);
    if (ok) {
        rowFinished();
    } else {
        rowFinished('Bad row: ' + row);
    }
},
function (error) {
    if (error) {
        console.log('Error:', error.message);
    } else {
        console.log('done processing');
    }
});
```

__Note:__ You need to manually close the cursor if you prematurely stop the iteration.
