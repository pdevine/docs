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

Lazily iterate over a cursor, array or feed one element at a time. `eachAsync` can be called with callback functions, or return a promise that will be resolved once all rows are returned.

The first, required function passed to `eachAsync` takes one or two arguments:

```js
// process each row asynchronously
cursor.eachAsync(function (row) {
    // function called for each element
});

// as above, but with a callback to execute when all rows are finished
cursor.eachAsync(function (row, function (err, data) {
    // function called when cursor is exhausted
}) {
    // function called for each element
});
```

The second function is another callback function which will be called if an error occurs.

```js
// process each row asynchronously
cursor.eachAsync(function (row) {
    // function called for each element
}, function (err) {
    // function called on error
});

__Example:__ Process all the elements in a stream, using `then` and `catch` for handling the end of the strem and any errors.

```js
cursor.eachAsync(function (row) {
    return process(row);
}).then(function () {
    console.log('done processing'); 
}).catch(function (err) {
    console.log('Error:', err);
});
```

__Example:__ As above, but using callbacks.

```js
var onDone = function (err, data) {
    if (err) {
        console.log('Error on finish:', err);
    } else {
        console.log('done processing');
    }
};

cursor.eachAsync(function (row, onDone) {
    return process(row);
},
function (err) {
    console.log('Error:', err);
});
```

__Example:__ Iteration can be stopped early by returning or throwing a promise that is rejected.

```js
cursor.eachAsync(function(row) {
    if (row.id < 10) {
        return process(row);
    } else {
        return Promise.reject();
    }
}).then(function () {
    console.log('done processing'); 
});
```

__Note:__ You need to manually close the cursor if you prematurely stop the iteration.
