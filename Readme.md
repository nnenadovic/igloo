
# Igloo

[![NPM version][npm-image]][npm-url]
[![Build Status][travis-image]][travis-url]
[![Dependency Status][depstat-image]][depstat-url]
[![NPM downloads][npm-downloads]][npm-url]
[![Test Coverage][coveralls-image]][coveralls-url]
[![MIT License][license-image]][license-url]

![Igloo][igloo-image]

> Igloo is a library of components built with [electrolyte][electrolyte] to be used as building blocks for an app.

Need to view documentation for a given component?  See [Components](#components) below.


## Index

* [Sponsor](#sponsor)
* [Install](#install)
* [Usage](#usage)
* [Components](#components)
* [Tests](#tests)
* [Conventions](#conventions)
* [Contributors](#contributors)
* [Credits](#credits)
* [License](#license)


## Sponsor

[![Clevertech][clevertech-image]](http://clevertech.biz)


## Install

**Please use [eskimo][eskimo] which has igloo built-in.**

However you can use igloo in your own project by installing it.

```bash
npm install -s igloo
```


## Usage

You will need to first `require('igloo')`, define a "config" component, and then use `electrolyte`'s `IoC.loader` method to load `igloo`'s components ([listed below][#components]).

> **Note that `igloo` requires you to have defined a "config" component (since `igloo`'s "setting" component requires it as a dependency).  For example, [Eskimo][eskimo] defines a "config" component [here][eskimo-default-config].**

```js
// boot/config.js

exports = module.exports = function() {
  return {
    defaults: {
      logger: {
        console: true,
        requests: true,
        mongo: false,
        file: false,
        hipchat: false
      },
      output: {
        handleExceptions: true,
        colorize: true,
        prettyPrint: false
      }
    },
    development: {
      port: 3000,
    },
    production: {
      port: 3080,
      requests: false,
      output: {
        colorize: false,
        prettyPrint: false
      }
    }
  }
}

exports['@singleton'] = true

```


```js
// app.js

var IoC = require('electrolyte')
var igloo = require('igloo')
var path = require('path')
var express = require('express')
var winstonRequestLogger = require('winston-request-logger')

IoC.loader(IoC.node(path.join(__dirname, 'boot')))
IoC.loader('igloo', igloo)

// logger component
var logger = IoC.create('igloo/logger')
logger.info('Hello world')

// settings component
var settings = IoC.create('igloo/settings')

// basic server

var app = express()

// winston request logger before everything else
// but only if it was enabled in settings
if (settings.logger.requests)
  app.use(winstonRequestLogger.create(logger))

app.get('/', function(req, res, next) {
  res.send("It's cold outside, but warm in the igloo")
})

app.listen(settings.port, function() {
  logger.info('Server started on port %d', settings.port)
})

```


## Components

* [error-handler](#error-handler)
* [knex](#knex)
* [logger](#logger)
* [mongo](#mongo)
* [mongoose-plugin](#mongoose-plugin)
* [server](#server)
* [sessions](#sessions)
* [settings](#settings)
* [update-notifier](#update-notifier)

### error-handler

> Returns [Express][express] route middleware for error handling.

[View source](lib/boot/error-handler.js)

```js
// example - error handler

var _ = require('underscore')
var errorHandler = IoC.create('igloo/error-handler')

app.post('/user', createUser, errorHandler)

function createUser(req, res, next) {

  if (!_.isString(req.body.name))
    return next({
      param: 'name',
      message: 'User name is missing'
    })

  // ... create a user in your db
  res.send('You successfully created a user')

}

```


### knex

> Returns a [Knex][knex] SQL database connection using `settings.knex`.

[View source](lib/boot/knex.js)

```js
// example - knex

var knex = IoC.create('igloo/knex')

app.get('/users', function(req, res, next) {
  knex
    .raw('SELECT * FROM USERS LIMIT 10')
    .exec(function(err, results) {
      if (err) return next(err)
      res.send(results)
    })
})

```


### logger

> Returns a [Winston][winston] logger object with pre-configured transports using `settings.logger` and `settings.output`.

[View source](lib/boot/logger.js)

```js
// example - logger

var logger = IoC.create('igloo/logger')

logger.info('Hello world')

```


### mongo

> Returns a [MongoDB][mongodb] NoSQL connection using `settings.logger`, `settings.output`, and `settings.mongo` and [Mongoose][mongoose] ORM.

> TODO: Should we rename this to `igloo/mongoose`?

[View source](lib/boot/mongo.js)

```js
// example - mongo

var validator = require('validator')
var mongoose = IoC.create('igloo/mongo')

var User = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    unique: true,
    validate: validator.isEmail
  }
})

mongoose.model('User', User)

```


### mongoose-plugin

> Returns a [Mongoose][mongoose] plugin helper for common schema additions.

[View source](lib/boot/mongoose-plugin.js)

```js
// example - mongoose plugin

var validator = require('validator')
var mongoose = IoC.create('igloo/mongo')
var mongoosePlugin = IoC.create('igloo/mongoose-plugin')

var User = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    unique: true,
    validate: validator.isEmail
  }
})

User.plugin(mongoosePlugin)

mongoose.model('User', User)

```


### server

> Returns a function which accepts a callback argument (currently no `err` argument is returned when this callback function is executed).  This component starts either a [cluster][cluster] (with one thread per CPU) or a simple single-threaded instance using `app.listen`.  This should be used with `app.phase` using [bootable][bootable], since bootable's "phases" (`app.phase`) method accepts this callback function as a standard argument when defining a "phase" (read more about bootable to understand this please).  This component uses `settings.logger`, `settings.output`, and `settings.server`.

[View source](lib/boot/server.js)

```js
// example - server

var bootable = require('bootable')
var express = require('express')

var app = bootable(express())

var server = IoC.create('igloo/server')
var logger = IoC.create('igloo/logger')

// this should always be the last phase
app.phase(server)

app.boot(function(err) {

  if (err) {
    logger.error(err.message)
    if (settings.showStack)
      logger.error(err.stack)
    process.exit(-1)
    return
  }

  logger.info('app booted')

})

```


### sessions

> Returns a [Redis][redis] connection using `settings.logger`, `settings.output`, and `settings.redis` with [express-session][express-session] and [connect-redis][connect-redis].

[View source](lib/boot/sessions.js)

```js
// example - sessions

var session = require('express-session')
var express = require('express')

var app = express()

var sessions = IoC.create('igloo/sessions')
var settings = IoC.create('igloo/settings')

// add req.session cookie support
settings.session.store = sessions
app.use(session(settings.session))

```


### settings

> Returns a settings object using a "config" component defined and loaded by [electrolyte][electrolyte] in a location such as "boot/config" (see initial example at the top of this Readme).

For a full config example file, see [Eskimo's default config][eskimo-default-config].

> TODO: Check into requiring a "local" component so Adnan's work is tested

[View source](lib/boot/settings.js)


### update-notifier

> Returns nothing, as it is used for notifying an app of module updates using [update-notifier][update-notifier].

> TODO: Look into why this isn't working per <https://github.com/yeoman/update-notifier/issues/25>

[View source](lib/boot/update-notifier.js)


## Tests

```bash
npm install -d
npm test
```


## Conventions

See [nifty-conventions][nifty-conventions] for code guidelines, general project requirements, and git workflow.


## Contributors

* Nick Baugh <niftylettuce@gmail.com>
* Adnan Ibrišimbegović <adibih@gmail.com>


## Credits

* [Igloo](http://thenounproject.com/term/igloo/26547/) by VALÈRE DAYAN from The Noun Project
* [ESKIMO IGLOO](http://www.colourlovers.com/palette/1933518/ESKIMO_IGLOO) (color palette)


## License

[MIT][license-url]


[license-image]: http://img.shields.io/badge/license-MIT-blue.svg?style=flat
[license-url]: LICENSE
[nifty-conventions]: https://github.com/niftylettuce/nifty-conventions
[npm-image]: http://img.shields.io/npm/v/igloo.svg?style=flat
[npm-url]: https://npmjs.org/package/igloo
[npm-downloads]: http://img.shields.io/npm/dm/igloo.svg?style=flat
[travis-url]: http://travis-ci.org/niftylettuce/igloo
[travis-image]: http://img.shields.io/travis/niftylettuce/igloo.svg?style=flat
[depstat-url]: https://gemnasium.com/niftylettuce/igloo
[depstat-image]: http://img.shields.io/gemnasium/niftylettuce/igloo.svg?style=flat
[coveralls-image]: https://img.shields.io/coveralls/niftylettuce/igloo.svg?style=flat
[coveralls-url]: https://coveralls.io/r/niftylettuce/igloo?branch=master
[igloo-image]: https://filenode.s3.amazonaws.com/igloo.png
[eskimo]: https://github.com/niftylettuce/eskimo
[clevertech-image]: http://eskimo.io/img/clevertech.png
[electrolyte]: https://github.com/jaredhanson/electrolyte
[knex]: http://knexjs.org
[express]: http://expressjs.com
[winston]: https://github.com/flatiron/winston
[mongodb]: https://github.com/mongodb/node-mongodb-native
[mongoose]: http://mongoosejs.com/
[bootable]: https://github.com/jaredhanson/bootable
[cluster]: http://nodejs.org/api/cluster.html
[redis]: http://redis.io
[express-session]: https://github.com/expressjs/session
[connect-redis]: https://github.com/visionmedia/connect-redis
[eskimo-default-config]: https://github.com/niftylettuce/eskimo/blob/master/boot/config.js
[update-notifier]: https://github.com/yeoman/update-notifier
