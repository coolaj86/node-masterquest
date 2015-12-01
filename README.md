Master Quest PostgreSQL
============

Master Quest is a brave new attempt at Data Mapping.

It kinda looks like an ORM, but it isn't. It's not SQL, it's NoSQL with benefits.

You get to choose which contraints to keep and which to forget.

* MasterQuest [for SQLite3](https://github.com/coolaj86/node-masterquest-sqlite3)
* MasterQuest [for PostgreSQL](https://github.com/coolaj86/node-masterquest-pg)

Guiding Principles

* NoSQL (well, as far as you care)
* Migrations don't suck
* Harness the awesomeness of indexes
* `deletedAt` to delete a record
* `null` to delete a field
* unindexed fields stored as json
* ids should be one either
  * deterministic (duplicate records not okay, last `updatedAt` wins)
  * cryptographically random (duplicate records consolidated in application)
* avoid mutating data (CRDT-ish style)
* if it won't scale, don't fret it (no foreign key contraints)
* `id` is simply named `id`
* multi-key ids should be `sha256sum(key1 + key2 + key3)`
* JavaScript is `camelCase`y, databases are `snake_case`y. We can handle that.
* join tables are in alphabet order i.e. `foo`, `bar`, `bar_foo`

TODO / In Progress

* Multi-Master Replication
* Relationships
  * currently detaches before saving (most important)
* MongoDB / RethinkDB -ish queries
* RealTime

USAGE
=====

**Install**

```bash
# sqlite3 (raspberry pi, digital ocean)
npm install --save 'https://github.com/coolaj86/node-masterquest-pg.git'

# postgresql (heroku, digital ocean)
npm install --save 'https://github.com/coolaj86/node-masterquest-sqlite3.git'
```

**Example**
```javascript
'use strict';

function loadMq(masterquest, db) {
  var schema = {
    modelname: 'Persons'
  , indices: [ 'firstName', 'lastName' ]
  , hasMany: [ 'children' ]
  };

  masterquest.wrap(db, schema).then(function (store) {

    // update (or create) deterministic record
    var john = {
      id: 'john.doe@email.com'
    , firstName: 'john'
    , lastName: 'doe'
    , dog: { name: 'ralph', color: 'gold' }
    , children: [ 'stacey@email.com' ]
    };

    //
    // Examples
    //

    store.Persons.upsert(john).then(function () {
      // note: if `dog` existed, it will be overwritten, not merged
      // note: `children` will be removed before save
    });

    store.Persons.get('john.doe@email.com').then(function (record) {
      // dog will be rehydrated from json
      // children will not be fetched and attached
      console.log(record);
    });

    store.Persons.find({ firstName: 'john' }, { limit: 5, orderBy: 'lastName' }).then(function (records) {
      // will find all records exactly matching 'john' for firstName
      console.log(record);
    });
  });
}
```

**Setup**
```javascript
function useSqlite3() {
  var mq = require('masterquest-sqlite3');
  var sqlite3 = require('sqlite3');
  var dbFile = '/tmp/data.sqlite3';
  var client = new sqlite3.Database(dbFile);

  loadMq(mq, client);
}

function usePostgreSql() {
  var dbUrl = 'postgres://username:password@localhost:5432/database';
  require('pg').connect(dbUrl, function (err, client, done) {
    var mq = require('masterquest-pg');

    loadMq(mq, client);
  });
}

if (process.env.DATABASE_URL) {
  // use PostgreSQL on Heroku
  usePostgreSql();
} else {
  // use SQLite3 on Raspberry Pi or Digital Ocean
  useSqlite3();
}
```

API
===

It's kinda CRUDdy... but don't let that scare you.

* `upsert(id, data)` - creates or updates based on existence in DB (use this)
  * modifies `createdAt` and or `updatedAt`
* `save(data)` - (just don't use this, please) creates or updates based on presence of ID
* `destroy(id)` - mark a record as `deletedAt` from DB
* `get(id)` - grab one by id
* `find(attrs, opts)` - grab many by indexable attributes
  * attrs
    * explicit `null` will find all (and requires that `limit` be set)
    * `{ foo: 2, bar: 6 }` will find records where `foo` is `2` *and* `bar` is `6`
  * opts
    * `orderBy`
    * `orderByDesc`
    * `limit`

Schema
======

Anything that isn't in the schema

* `indices` specifies an array of strings
  * `[ 'firstName', 'lastName' ]`
* relationships are option and current only exclude during save
  * `hasMany`, `belongsTo`, `hasOne`, `belongsToMany`, `hasAndBelongsToMany`
* `createdAt`, `updatedAt`, `deletedAt` timestamps are always added
  * turn off with `timestamps: false`
* `id` is always `id`
  * change with `idname: 'myId'`

Migrations
----------

You can only add indexes. You cannot rename or remove them.

To add an index, simply change the schema.

### For example

If you had this schema

```javascript
{ modelname: 'Persons'
, indices: [ 'firstName', 'lastName' ]
, hasMany: [ 'children' ]
}
```

And you then created this schema:

```javascript
{ modelname: 'Persons'
, indices: [ 'firstName', 'lastName', 'favoriteColor' ]
, hasMany: [ 'children' ]
}
```

Then `favoriteColor` would be added as an index.


LICENSE
=======

Dual-licensed MIT and Apache-2.0

See LICENSE
