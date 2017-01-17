# JavaScript Documentation

This is the basic Node.js JavaScript API.

## Index

* [Getting Started](#first-link)
* [API](#second-link)
* [Examples](#third-link)

## Getting Started
TODO

## API
### Container
This is the highest level module that wraps IDO below.  Naming TBD
#### `initialize(serverUrl, apiKey, options)`
+ This method must be called first to initialize the library.

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

#### `create(content, options)` -> `'containerId'`

   Creates an encrypted container with the provided `content`. The container will be uploaded and access will be granted to the specified users, unless the `localAccessOnly` option is set to `true`.

   Returns a Promise that resolves to the new container's ID.

   Throws an Error if the connection is unavailable or an access userId is not found.

Parameter   | Type  | Description
:------|:------|:-----------
content | Buffer | Node.js Buffer for the data to be stored in the container.
options | Object | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
access | Array of [AccessInfo](#AccessInfo) | `[]` if not defined in initialize options | The access granted to the container on upload.
header | Object | `{}` | Use this to store any metadata about the content.  This data is independently encrypted and can be retrieved prior to downloading and decrypting the full content.
localAccessOnly | boolean | `false` | This prevents uploading the container.  The container will only be accessible locally.
type | String | TODO define `'default type'` | A string used to categorize the container on the server.

#### `update(id, options)`
+ Updates the container with the specified ID. At least one optional parameters must be provided for an update to occur.

+ Returns a Promise.

+ Throws an Error if the connection is unavailable or an access userId is not found.

Parameter   | Type  | Description
:------|:------|:-----------
id | String | The ID of the container to update
options | Object | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
|||TODO Add all rows from `create` when finalized
content | Buffer | null | The content to update.

#### `getDecrypted(id, options)` -> [DecryptedContainer](#DecryptedContainer)
+ Gets the container and decrypts it for usage. By default it downloads any required data, includes the content, and caches any downloaded data locally.  See options for overriding this behavior.

+ Returns a Promise that resolves to a [DecryptedContainer](#DecryptedContainer)

+ Throws an Error if the container or connection is unavailable.

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

### Utilities
#### `hash(seed)` -> 'hashedString'
+ Produces a sha256 hash of the specified seed.

+ Returns a string with the hashed value.

Parameter   | Type  | Description
:------|:------|:-----------
seed | String | Seed for producing the hash

## Examples
TODO
