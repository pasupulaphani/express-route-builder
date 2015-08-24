# Express Route Builder

When express apps grow beyond a hack day project they often end up with a long middleware chain such as:

* authentication - user logged in?
* authorization - user allowed access?
* content-type checks - appropriate content type sent?
* input validation/sanitisation - ensure bad values don't get through
* route handler (controller) - do some work
* output validation/sanitisation - ensure secret values don't get out
* pagination headers
* dispatchers - respond to json/html/xml/csv based on data
* error handling

This can turn into a bit of a mess of registering global/route specific middleware.

[Express Route Builder](https://github.com/oliverbrooks/express-middelware-builder) is here to help simplify building those middleware chains. 

## Objectives

* DRY middleware configuration
* Configure middleware routines via objects, not re-writing code
* handle errors gracefully through deep middleware stacks

## Usage

1. Make some middleware generators.

These are generator functions which take values from the route objects and return configured middleware.

The idea behind middleware generators is to provide an easily pluggable architecture for extending the functionality of middlewares on a per-route basis.  These generators can be easily modularised in separate NPM repos.

`./middlewares.js`

```js
var expressRouteBuilder = require("express-route-builder");

/*
 * Middleware generators: Functions that return a middleware.
 * The generator will be passed any values from route objects
 */
function paginatorBefore (valueFromRoute) {
    return function (req, res, next) {
        // ... set some pagination defaults
    }
}

function authUser (valueFromRoute) {
    return function (req, res, next) {
        // ... fetch user from cookie etc
    }
}

function handler (controllerFromRoute) {
    return function (req, res, next) {
        // Nice way to handle errors if handlers return a promise
        return controllerFromRoute(req, res)
            .then(function (body) {
                res.body = body;
            })
            .nodeify(next);
    }
}

function paginatorAfter (valueFromRoute) {
    return function (req, res, next) {
        // ... set some pagination headers
    }
}

function dispatchers (accepts) {
    return function (req, res, next) {
        var acceptHeader = req.get("acccept");
        if (acceptHeader === "application/json") {
            res.json(res.body);
        } else if (acceptHeader === "text/html") {
            res.send(res.body);
        }
    }
}

/*
 * Route builder configuration
 * These are run in the order of this object
 */
expressRouteBuilder.setMiddlewares([
    {
        // name is used in logging and to identify in route object
        name: "paginatorBefore",
        // 'get' means it will be included in all HTTP GET requests
        include: "get",
        generator: paginatorBefore
    },
    {
        name: "authUser",
        // 'optional' means it must be set in the route object to include
        include: "optional",
        generator: authUser
    },
    {
        name: "handler",
        // 'required' means it must be specified or an error is thrown
        include: "required",
        generator: handler
    },
    {
        name: "paginatorAfter"
        include: "get",
        generator: paginatorAfter
    },
    {
        name: "dispatchers",
        // 'all' means the middleware will always be included
        include: "all",
        generator: dispatchers
    }
]);

```

2. Create a routes configuration file

These are plain JavaScript objects whose keys match the names of the middlewares registered with express-route-builder.


`./badger_routes.js`

```js


/*
 * Routes will be:
 * GET /badgers
 * middlewares: paginatorBefore, authUser, handler, paginatorAfter, dispatcher
 *
 * POST /badgers
 * middlewares: authUser, handler, dispatcher
 */

module.exports = [
    {
        method: "get",
        path: "/badgers",
        authUser: "session",
        handler: function (req, res) {
            // Fetch some badgers
        }
    },
    {
        method: "post",
        path: "/badgers",
        authUser: "session",
        handler: function (req, res) {
            // Make more badgers
        }
    },
]

```

3. Create an express server

Normal express server. Build routes using express-route-builder and can mount router as per usual.

`./server.js`

```js
var middlware = require("./middleware");
var express = require("express");
var badgerRoutes = require("./badger_routes")

/*
 * Server setup
 */
var app = express();

var badgerRouter = expressRouteBuilder.buildRouter(badgerRoutes);

// Routes
app.use("/animals", badgerRouter);

// Error handling
app.use(function (err, req, res, next) {
    // do something with next(err) errors
});

app.listen(3000, function (err) {
    if (!err) {
        console.log("server running");
    }
});
```


## Documentation

See `lib/index`

## Development

Tests run using mocha or via `npm test`.

To debug middleware generators in your app express-route-builder uses [node-debug]() and logging can be seen with `DEBUG=middleware:* npm start`. This will print all the middleware the app has passed through.