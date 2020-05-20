# east

east - node.js database migration tool for different databases (extensible via
[adapters](#adapters)).

east connects to the db using particular adapter (mongodb, sqlite, postgres,
mysql, couchbase), keeps track of executed migrations by storing their names
inside db and makes connect to the db available inside `migrate` and `rollback`
functions. east encourages you to use for migrations driver/syntax with which
you are already familiar with (apparently you use it for work with db at your
application) and doesn't provide universal api for working with any kind of
database.

[![Npm version](https://img.shields.io/npm/v/east.svg)](https://www.npmjs.org/package/east)
[![Build Status](https://travis-ci.org/okv/east.svg?branch=master)](https://travis-ci.org/okv/east)
[![Coverage Status](https://coveralls.io/repos/github/okv/east/badge.svg)](https://coveralls.io/github/okv/east)
[![Known Vulnerabilities](https://snyk.io/test/npm/east/badge.svg)](https://snyk.io/test/npm/east)


Following subjects described below:

* [Node.js compatibility](#nodejs-compatibility)
* [Installation](#installation)
* [Changelog](#changelog)
* [Cli usage](#cli-usage)
* [Library usage](#library-usage)
* [Adapters](#adapters)
* [Plugins](#plugins)
* [Creating own adapter](#creating-own-adapter)
* [License](#license)


## Node.js compatibility

east itself requires node.js >= 4 to work.

Please note that particular adapter may have another requirements (see
documentation for specific adapter).


## Installation

```sh
npm install east -g
```

alternatively you could install it locally


## Changelog

All notable changes to this project will be documented in
[CHANGELOG.md](CHANGELOG.md).


## Cli usage

At your project dir run

```sh
east init
```

after that you can `create`, `migrate`, `rollback` your migrations.

Run `east -h` to see all commands:

```sh

  Usage: east [options] [command]

  Commands:

    init                   initialize migration system
    create <basename>      create new migration based on template
    migrate [options]      run all or selected migrations
    rollback [options]     rollback all or selected migrations
    list [status]          list migration with selected status (`new`, `executed` or `all`), `new` by default
    *

  Options:

    -h, --help           output usage information
    -V, --version        output the version number
    --adapter <name>     which db adapter to use
    --config <path>      config file to use
    --timeout <timeout>  timeout for migrate/rollback
    --template <path>    path to template for new migrations
    --dir <dir>          dir where migrations stored
    --url <url>          db connect url
    --trace              verbose mode (including error stack trace)

```

run `east <command> -h` to see detailed command help.

All options described above can be set via command line or at `.eastrc` file
located at current directory, e.g.:

```js

{
	"dir": "./dbmigration",
	"template": "./lib/node/utils/customMigrationTemplate.js"
}

```

`.eastrc` also can be a regular nodejs script (instead of json file):

```js

var path = require('path');

module.exports = {
    dir: path.join(__dirname, 'dbmigration'),
    template: './lib/node/utils/customMigrationTemplate.js'
};

```


### create

```sh
east create doSomething
```

produces something like this

```sh
New migration `1_doSomething` created at migrations/1_doSomething.js
```

created file will contain

```js
exports.migrate = function(client, done) {
    done();
};

exports.rollback = function(client, done) {
    done();
};
```

* `client` is connect to current db and he determined by adapter (see [adapters](#adapters) section)
* `done` is function which should be called at the end of migration (if any
error occured you can pass it as first argument)
* instead of using `done` argument promise can be returned or async function can be used
* migration also can be synchronous - declare only `client` at `migrate` or `rollback`
* `rollback` function is optional and may be omitted

Migration file is regular node.js module and allows migrate any database e.g.

```js
// include your database wrapper which you already use in app
var db = require('./db');

exports.migrate = function(client, done) {
    db.connect(function(err) {
        if (err) done(err);
        db.things.insert({_id: 1, name: 'apple', color: 'red'}, done);
    });
};

exports.rollback = function(client, done) {
    db.connect(function(err) {
        if (err) done(err);
        db.things.remove({_id: 1}, done);
    });
};

```

or you can use special adapter for database (see [adapters](#adapters) section)


#### Migration file number format

The default format for migration file names is to prepend a number to the
filename which is incremented with every new file. This creates migration files
such as `migrations/1_doSomething.js`, `migrations/2_doSomethingElse.js`.

If you prefer your files to be created with a date time instead of sequential
numbers, you can set the `migrationNumberFormat` configuration parameter in
your `.eastrc` to `dateTime`:

```json
{
    "migrationNumberFormat": "dateTime"
}
```

This will create migration files with date time prefix in `YYYYMMDDhhmmss`
format (e.g. `migrations/20190720172730_doSomething.js`).

For the default behaviour, you can omit the `migrationNumberFormat`
configuration option or set it to:

```json
{
    "migrationNumberFormat": "sequentialNumber"
}
```


### migrate

let's create one more migration

```sh
east create doSomethingElse
```

then executes both of them

```sh
east migrate
```

it sequentially executes all new migrations and produces

```sh
target migrations:
    1_doSomething
    2_doSomethingElse
migrate `1_doSomething`
migration done
migrate `2_doSomethingElse`
migration done
```

selected migrations can be executed by passing their names (or numbers or
basenames or paths) as argument

```sh
east migrate 1_doSomething 2
```

in our case this command will skip all of them

```sh
skip `1_doSomething` because it`s already executed
skip `2_doSomethingElse` because it`s already executed
nothing to migrate
```

you can pass `--force` option to execute already executed migrations.
This is useful while you develop and test your migration.

You also can export `tags` array from migration and then migrate only
migrations that satisfied expression specified by option `--tag`. Expression
consists of tag names and boolean operators `&`, `|` and `!`. For example,
following command will migrate all migrations that have tag `tag1` and not have
tag `tag2`:

```sh
east migrate --tag 'tag1 & !tag2'
```


### rollback

`rollback` has similar to `migrate` command syntax but executes `rollback`
function from migration file

```sh
east rollback
```

will produce

```sh
target migrations:
    2_doSomethingElse
    1_doSomething
rollback `2_doSomethingElse`
migration successfully rolled back
rollback `1_doSomething`
migration successfully rolled back
```


### list

```sh
east list
```

shows new migrations e.g.

```sh
new migrations:
     1_doSomething
     2_doSomethingElse
```

target status could be specified as an argument e.g.

```sh
east list executed
```


## Library usage

east exposes `MigrationManager` class (descendant of `EventEmitter`) which for
example can be used to migrate your database from node.js app without using
east cli:

```js
const {MigrationManager} = require('east');

const main = async () => {
    const migrationManager = new MigrationManager();

    // log target migrations before execution
    migrationManager.once('beforeMigrateMany', (migrationNames) => {
        console.log('Target migrations: ', migrationNames);
    });

    await migrationManager.configure();

    try {
        await migrationManager.connect();
        // select for migration all not executed migrations
        await migrationManager.migrate({status: 'new'});
    }
    finally {
        await migrationManager.disconnect();
    }
}

main().catch((err) => {
    console.error('Some error occurred: ', err.stack || err);
});
````

`MigrationManager` methods:

* **configure(params)** - configures migration process, accepts object with
parameters (`dir`, `adapter`, etc) and merges it with loaded config (when
`loadConfig` param is truthy - true by default). Returns *Promise<void>*. **This
method should be called before any other methods.**

* **getParams()** - returns *Promise* with parameters used by migration
process after configuration(`configure` method).

* **init()** - initiates migration process for a project. Should be called once
per project. Returns *Promise<void>*.

* **isInitialized()** - checks whether `init` was made or not.
Returns *Promise<Boolean>*.

* **create(basename)** - creates migration, returns *Promise* with migration
object.

* **getMigrationPath(name)** - returns absolute path of the migration on disk
by name of the migration. Returns *Promise<String>*.

* **connect()** - connects to database management system (if supposed by
adapter). Returns *Promise<void>*.

* **getMigrationNames({migrations, status, tag, reverseOrderResult})** -
returns migrations names, following options are provided:

  * **migrations** - array of target migrations, each migration could be
  defined by basename, full name, path or number.
  * **status** - status to filter migrations, supported statuses are:
  `new`, `executed` and `all`.
  * **tag** - tag expression to filter migrations e.g. `'tag1 & !tag2'`
  * **reverseOrderResult** - if true then result array will be reversed.

`migrations` and `status` are mutually exclusive.
If `migrations`, `status` not provided then all migrations will be processed
(e.g. filtered by tag and returned).

* **migrate({migrations, status, tag, force})** - executes target migrations.
Target migration could be defined by `migrations`, `status`, `tag` options
(see it's description at `getMigrationNames` method). *By default*
migrations with status `new` are chosen. Returns *Promise<void>*. `force`
flag allows to execute already executed migrations.

* **rollback({migrations, status, tag, force})** - rollbacks target migrations.
Target migration could be defined by `migrations`, `status`, `tag` options
(see it's description at `getMigrationNames` method). *By default*
migrations with status `executed` are chosen. Returns *Promise<void>*. `force`
flag allows to rollback not executed migrations.

* **disconnect()** - disconnects from database management system (if supposed
by adapter). Returns *Promise<void>*.


`MigrationManager` events:

* **beforeMigrateOne({migration})**
* **afterMigrateOne({migration})**
* **beforeMigrateMany({migrationNames})**
* **afterMigrateMany({migrationNames})**
* **beforeRollbackOne({migration})**
* **afterRollbackOne({migration})**
* **beforeRollbackMany({migrationNames})**
* **afterRollbackMany({migrationNames})**
* **onSkipMigration({migration, reason})**


## Adapters

adapter determines where executed migration names will be stored and what will be
passed to `migrate` and `rollback` function as `client`.
Default adapter store executed migration names at file `.migrations` which is
located at migrations directory and pass `null` as `client`.

Other adapters:
* [mongodb](https://github.com/okv/east-mongo)
* [sqlite](https://github.com/2do2go/east-sqlite)
* [postgres](https://github.com/2do2go/east-postgres)
* [mysql](https://github.com/riggerthegeek/east-mysql)
* [couchbase](https://github.com/ramiel/east-couchbase)
* [couchdb](https://github.com/schipiga/east-couchdb)


## Plugins

East functionality could be extended by using plugins, for usage instructions
see plugin page:

* [migration duration logger](https://github.com/okv/east-migration-duration-logger)
* [migration progress indicator helper](https://github.com/okv/east-migration-progress-indicator-helper)


## Creating own adapter

For writing your own adapter you should implement methods for connection,
mark transaction as executed, etc see details inside
[built-in adapter](lib/adapter.js) and [other adapters](#adapters).

You also can run migrator tests from current repository against your adapter:

* Clone current repository
* Change current directory to it
* Create file `.eastrc` with path and parameters for
your adapter e.g.

```js
{
    "adapter": "../../east-mysql/lib/adapter",
    "url": "mysql://user:password@localhost/east_test_db",
    "createDbOnConnect": true
}
```

* Run `NODE_EAST_TEST_LOAD_CONFIG=1 npm run testSpecified test/01-migrator -- --jobs=1` at
root of the cloned repository.


## License

MIT
