# radiks-server

![radiks.js](./radiks.js@2x.png)

<!-- TOC depthFrom:2 -->

- [radiks-server](#radiks-server)
  - [Introduction](#introduction)
    - [Privacy](#privacy)
    - [Multi-user scenarios](#multi-user-scenarios)
    - [Authorization](#authorization)
  - [Use with built-in CLI server](#use-with-built-in-cli-server)
    - [Running a custom Radiks-server](#running-a-custom-radiks-server)
    - [Accessing the MongoDB Collection](#accessing-the-mongodb-collection)
      - [Using `getDB` to manually connecting to the MongoDB collection](#using-getdb-to-manually-connecting-to-the-mongodb-collection)
      - [Migration from Firebase (or anywhere else)](#migration-from-firebase-or-anywhere-else)
    - [Options](#options)

<!-- /TOC -->

## Introduction

Radiks-server is a pre-built server to index and serve data that lives in decentralized services. Specifically, it is built to index data that is stored in [Gaia](https://github.com/blockstack/gaia), and created using the front-end companion library, [radiks.js](https://github.com/hstove/radiks).

Because Gaia is just a key-value store, many applications end up needing to store an 'index' of all underlying data, in order to query it in a performant and flexible way.

### Privacy

Radiks is designed to store highly private information. Because radiks.js encrypts all sensitive data before it ever leaves the client, this server ends up being a 'dumb' database that simply stores all encrypted data in an easily-queryable format.

This means that the server is only able to return queries for unencrypted data. Radiks.js is designed to be able to query for non-private information, and then decrypt the sensitive information on the client.

### Multi-user scenarios

Many decentralized apps include publicly sharable information that is created with the intent of sharing that data with the world. In these situations, you want to be able to query across all user's data, using complicated queries like text search, joins, and filters.

For example, consider a Twitter-clone app. You have many different users creating their own tweets, and those tweets are stored in their own storage backends. This ensures that the user has full control and ownership of their data. You still need a central server to keep track of everyone's tweets, so that you can serve combined timelines and perform searches. Radiks-server excels in this scenario.

### Authorization

Although Radiks-server is mostly a 'dumb' database that stores an index of decentralized data, there are specific authorization rules that only allow writes with the correct demonstration that the user owns the data they're writing.

Radiks.js creates and manages 'signing keys' that it uses to sign all writes that a user performs. Radiks-server is aware of this mechanism, and validates all signatures before performing a write. This guarantees that a user is not able to over-write a different user's data.

Radiks-server also is built to support writes in a collaborative, but private, situation. For example, consider a collaborative document editing application, where users can create 'organizations' and invite users to that organization. All users in that organization have read and write priveleges to data related to that organization. These organizations have single 'shared key' that is used to sign and encrypt data. When an organization administrator needs to remove a user from the group, they'll revoke a previous key and create a new one. Radiks is aware of these relationships, and will only support writes that are signed with the current active key related to a group.

## Use with built-in CLI server

Radiks-server is a `node.js` application that uses [MongoDB](https://www.mongodb.com/) as an underlying database. The easiest way to run `radiks-server` is to use the pre-packaged `node.js` server that is included with this `npm` package. 

In the future, Radiks-server will support various different databases, but right now only MongoDB is supported. 

1. Install and run MongoDB 3.6 or higher.

   **You must use MongoDB >=3.6, because they fixed an issue with naming patterns in keys.**

   If you are testing on a local workstation, you can install locally or use a `docker` image.  To install, visit their [download page](https://www.mongodb.com/download-center/community). You can also download MongoDB using your favorite package manager. On Mac OS, [homebrew is recommended](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-os-x/#install-mongodb-community-edition-with-homebrew).

2. On your MongoDB instance, create a database for your application data.
  
3. Create a username/password combination with `root` privileges on your new database.

4. Install the `radiks-server` on a workstation or server.

    ```bash
    npm install -g radiks-server
    ```
    Or, if you prefer `yarn`:

    ```bash
    yarn global add radiks-server
    ```

5. Create an `MONGODB_URI` environment variable on the same machine where you are running the `radiks-server`.

   Use the `mongodb://username:password@host:port/db_name` format for your variable. For example, to set this variable in a `bash` shell:

   ```bash
   export MONGODB_URI="mongodb://admin:mongome@157.245.167.8:27017/mycoolapp"
   ```

   The default port for Mongodb is `27017`, your instance may be configured differently.  By default, Radiks-server will use `'mongodb://localhost:27017/radiks-server'` as the `MONGODB_URI` value. This is suitable for local testing, but in production, you'll want to change the hostname and possible the database name.

6. Run `radiks-server` in the command line to start a server. 

   The `radiks-server` defaults to running on port `1260`, but you can use the `PORT` environment variable to modify this.

7. Configure your application to use your `radiks-server`.
  
    To configure your applciation as a `radiks` client, use code that looks like this when starting up your application:

    ```js
     import { UserSession, AppConfig } from 'blockstack';
     import { configure } from 'radiks';

     const userSession = new UserSession({
       appConfig: new AppConfig(['store_write', 'publish_data'])
     })

     configure({
       apiServer: 'http://my-radiks-server.com',
       userSession
     });
    ```
 
   For more information on configuring and writing a Radiks a client application, see [the Radiks client](https://github.com/blockstack-radiks/radiks) repository. 

8. Build and run your application.


### Running a custom Radiks-server

If you're using an [express.js](https://expressjs.com/) server to run your application, it's probably easiest to use the Radiks-server middleware. This way, you won't have to run a separate application server and Radiks server.

Radiks-server includes an easy-to-use middleware that you can include in your application:

```javascript
const express = require('express');

const { setup } = require('radiks-server');

const app = express();

setup().then(RadiksController => {
  app.use('/radiks', RadiksController);
});
```

The `setup` method returns a promise, and that promise resolves to the actual middleware that your server can use. This is because it first connects to MongoDB, and then sets up the middleware with that database connection.

The `setup` function accepts an `options` object as the first argument. Right now, the only option supported is `mongoDBUrl`. If you aren't using environment variables, you can explicitly pass in a MongoDB URL here:

```javascript
setup({
  mongoDBUrl: 'mongodb://localhost:27017/my-custom-database',
}).then(RadiksController => {
  app.use('/radiks', RadiksController);
});
```

### Accessing the MongoDB Collection

#### Using `getDB` to manually connecting to the MongoDB collection

Radiks-server keeps all models inside of a collection. You can use the `getDB` function to access this collection. [See the MongoDB Collection reference](https://mongodb.github.io/node-mongodb-native/3.1/api/Collection.html) for documentation about how you can interact with this collection.

```js
const { getDB } = require('radiks-server');

const mongo = await getDB(MONGODB_URL);
```

#### Migration from Firebase (or anywhere else)

Migrating data from Firebase to Radiks-server is simple and painless. You can create a script file to fetch all the firebase data using their API. Then, you can use your MONGOD_URI config to use the `mongodb` npm package.

```js
// Script for transfering users from Firebase to Radiks-server

const { getDB } = require('radiks-server');
const { mongoURI } = require('......'); // How you import/require your mongoURI is up to you

const migrate = async () => {
  // `mongo` is a reference to the MongoDB collection that radiks-server uses.
  // You can add or edit or update data as necessary.
  const mongo = await getDB(mongoURI);

  /**
   * Call code to get your users from firebase
   * const users = await getUsersFromFirebase();
   * OR grab the Firebase JSON file and set users to that value
   * How you saved your user data will proably be different than the example below
   */

  const users = {
    '-LV1HAQToANRvhysSClr': {
      blockstackId: '1N1DzKgizU4rCEaxAU21EgMaHGB5hprcBM',
      username: 'kkomaz.id',
    },
  };

  const usersToInsert = Object.values(users).map(user => {
    const { username } = user;
    const doc = {
      username,
      _id: username,
      radiksType: 'BlockstackUser',
    };
    const op = {
      updateOne: {
        filter: {
          _id: username,
        },
        update: {
          $setOnInsert: doc,
        },
        upsert: true,
      },
    };
    return op;
  });

  await mongo.bulkWrite(usersToInsert);
};

migrate()
  .then(() => {
    console.log('Done!');
    process.exit();
  })
  .catch(error => {
    console.error(error);
    process.exit();
  });
```

### Options

You can specify some options while initiating the Radiks server.

```javascript
const { setup } = require('radiks-server');

setup({
  ...myOptions,
});
```

Available options:

- `mongoDBUrl` - The MongoDB URL for the Radiks server
- `maxLimit` - The maximum `limit` field used inside the mongo queries - default to 1000
