# JavaScript Documentation

This is the basic Node.js JavaScript API.

## Index

* [Getting Started](#first-link)
* [Examples](#third-link)
* [API](#second-link)

## Getting Started
1. Installation:
   ```
   npm install absio
   ```
2. Import as regular module and initialize:

   ``` javascript
   var absio = require('absio');

   ```
3. Initialize the library and log in with an account:   
   ``` javascript
   absio.initialize('api.absio.com', yourApiKey);
   await absio.logIn('ed46da09-40dc-45c4-9c1a-8c5e11334986', accountPassword, accountAnswer);
   ```
4. Start creating encrypted containers:
   ``` javascript
   const sensitiveData = new Buffer('information to protect');
   const containerAccess = [{
     userId: 'ed46da09-40dc-45c4-9c1a-8c5e11334986',
     permission: 'read-write',
     expiration: new Date(2020)
   }];

   const containerId = await absio.create(sensitiveData, { access: containerAccess });
   ```

## Examples
TODO

## API
TODO - API Index
### Container
This is the highest level module that wraps IDO below.  Naming TBD
### `initialize(serverUrl, apiKey, options)`
This method must be called first to initialize the library.

Parameter   | Type  | Description
:------|:------|:-----------
serverUrl | String | The URL of the API server. (Example TODO: sandbox.absio.com)
apiKey | String | The API Key for your Absio Development Account ([TODO link](https://developer.absio.com/register))
options | Object | See table below

Option | Type  | Default | Description
:------|:------|:--------|:-----------
cacheLocal | boolean | `true` | Set false to prevent caching information in local database and OFS
defaultAccess | Array of [AccessInfo](#AccessInfo) | `[]` | This defines the default access for all methods that grant access to objects.
##### AccessInfo
```javascript
{
  expiration: <null or Date()>,
  permissions: <"read", "read-write", "write">,
  userId: 'userIdOfUserWithDefaultAccess'
}
```

---
### `logIn()`
+ TODO

---
### `create(content, options)` -> `'containerId'`

   Creates an encrypted container with the provided `content`. The container will be uploaded and access will be granted to the specified users, unless the `localAccessOnly` option is set to `true`.

   Returns a Promise that resolves to the new container's ID.

   Throws an Error if the connection is unavailable or an access userId is not found.

Parameter   | Type  | Description
:------|:------|:-----------
`content` | Buffer | Node.js Buffer for the data to be stored in the container.
`options` | Object | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`access` | Array of [AccessInfo](#AccessInfo) | `[]` if not defined in initialize options | The access granted to the container on upload.
`header` | Object | `{}` | Use this to store any metadata about the content.  This data is independently encrypted and can be retrieved prior to downloading and decrypting the full content.
`localAccessOnly` | boolean | `false` | This prevents uploading the container.  The container will only be accessible locally.
`type` | String | TODO define `'default type'` | A string used to categorize the container on the server.

---
### `update(id, options)`
Updates the container with the specified ID. At least one optional parameters must be provided for an update to occur.

Returns a Promise.

Throws an Error if the connection is unavailable or an access userId is not found.

Parameter   | Type  | Description
:------|:------|:-----------
id | String | The ID of the container to update
options | Object | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
|||TODO Add all rows from `create` when finalized
content | Buffer | null | The content to update.

---
### `getDecrypted(id, options)` -> [DecryptedContainer](#DecryptedContainer)
Gets the container and decrypts it for usage. By default it downloads any required data, includes the content, and caches any downloaded data locally.  See options for overriding this behavior.

Returns a Promise that resolves to a [DecryptedContainer](#DecryptedContainer)

Throws an Error if the container or connection is unavailable.

Parameter   | Type  | Description
:------|:------|:-----------
id | String | The ID of the container to update
options | Object | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
cacheLocal | boolean | `true` | Set false to prevent caching information in local database and OFS
includeContent | boolean | `true` | Set to `false` to prevent downloading and decrypting content.  This is helpful when the content is very large.
localAccessOnly | boolean | `false` | Prevents downloading container from the server. Only locally cached containers will be available.

##### DecryptedContainer
``` javascript
{
  access: [
    {
      expiration: <null or Date()>,
      permissions: <"read", "read-write", "write">,
      userId: 'userIdWithAccess'
    },
    ...
  ]
  content: Buffer()
  header: {},
  id: 'IdAsGuid',
  length: 12345, // TODO What length is this?
  storageInformation: {
    created: Date(),
    filePath: 'Path in local file system',
    latestEventId: 543,
    modified: Date(),
    modifiedBy: 'userId that modified',
    ownerId: 'userId of owner',
    type: 'containerType',
    url: 'URL to download container',
    urlExpiration: Date()
  }
}
```

---
### `getLatestByType(type, options)` -> `[{ decryptedContainer }]`
Downloads and decrypts any new or updated containers of the specified type. This will return all new containers since the last call of this method, unless specified in `options`.

Returns a Promise that resolves to an Array of [DecryptedContainer](#DecryptedContainer).

Throws an Error if the connection is unavailable.

Parameter   | Type  | Description
:------|:------|:-----------
type | String | A string used to categorize the container on the server.
options | Object | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
startingEventId | Number | -1 | 0 will start from the beginning and download all containers for the current user.  Use the `storageInformation.latestEventId` field of the [DecryptedContainer](#DecryptedContainer) to start from existing successful event. -1 will download all new since last call.

---
### `delete(id, options)`
Deletes the container from the server and local file system, unless specified in options.

Returns a Promise.

Parameter   | Type  | Description
:------|:------|:-----------
id | String | The ID of the container to delete
options | Object | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
localAccessOnly | boolean | `false` | Prevents deleting container from the server. Only locally cached containers will be deleted.

---
### Utilities
### `hash(seed)` -> 'hashedString'
Produces a sha256 hash of the specified seed.

Returns a string with the hashed value.

Parameter   | Type  | Description
:------|:------|:-----------
seed | String | Seed for producing the hash
