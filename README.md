# Absio Secured Container

## Index

* [Overview](#overview)
* [Getting Started](#getting-started)
* [Usage](#usage)
* [API](#api)

## Overview
TODO - general

### Users
TODO

### Encryption
TODO

## Getting Started
The `userId`, `password`, and `answer` used below in Step 2 are the credentials for an existing user. [Users](#users) can be created with the `register()` method or with our web-based secure user creation utility. For more details see the [Users](#users) section above.

1. Installation:

   ```
   npm install absio-secured-container
   ```
2. Import as regular module and initialize:

   ``` javascript
   var securedContainer = require('absio-secured-container');
   ```
3. Initialize the library and log in with an account:  

   ``` javascript
   securedContainer.initialize('your.absioApiServer.com', yourApiKey);
   await securedContainer.logIn('ed46da09-40dc-45c4-9c1a-8c5e11334986', accountPassword, accountAnswer);
   ```
4. Start creating secured containers:

   ``` javascript
   const sensitiveData = new Buffer('information to protect');
   const containerAccess = [{
       userId: 'ed46da09-40dc-45c4-9c1a-8c5e11334986',
       permission: 'read-write',
       expiration: new Date(2020)
   }];

   const containerId = await securedContainer.create(sensitiveData, { access: containerAccess });
   ```

## Usage
The following usage examples requires that the general setup in [Getting Started](#getting-started) has been completed.

TODO - General example here


### Possible 418 Usage
Below are three examples specific to our understanding of the simplest usage in the 418 use cases.

#### Customer System

``` javascript
async function shareUnstructuredData(unstructuredData) {
    const containerOptions = {
        access: [trustedDataBrokerID],
        type: 'unstructured-data'
    };

    await securedContainer.create(unstructuredData, containerOptions);
}
```

#### Trusted Data Broker

``` javascript
async function processUnstructuredData() {
    const unstructuredDataContainers = await securedContainer.getLatestByType('unstructured-data');

    for (let container of unstructuredDataContainers) {
        const results = createNetFlowDataWithDataMap(container.content);

        await shareNetFlowData(results.netFlowFormat);
        await shareObfuscatedDataMap(results.obfuscatedDataMap);
    }
}

async function shareNetFlowData(netFlowData) {
    const containerOptions = {
        access: [analysisSystemID],
        type: 'net-flow-data'
    };

    await securedContainer.create(netFlowData, containerOptions);
}

async function shareObfuscatedDataMap(obfuscatedDataMap) {
    const containerOptions = {
        access: [customerSystemID],
        type: 'obfuscated-data-map'
    };

    await securedContainer.create(obfuscatedDataMap, containerOptions);
}
```

#### Analysis System

``` javascript
async function processNetFlowData() {
    const netFlowContainers = await securedContainer.getLatestByType('net-flow-data');

    for (let container of netFlowContainers) {
        const report = performAnalysis(container.content);

        await securedContainer.create(report, {
            access: [trustedDataBrokerID],
            type: 'report'
        });
    }
}

async function processUpdatedReports() {
    const updatedReportContainers = await securedContainer.getLatestByType('reports', { updatesOnly: true });

    for (let reportContainer of updatedReportContainers) {
        updateSystemForReport(reportContainer.content);
    }
}
```

## API
TODO - API Index

### `initialize(serverUrl, apiKey[, options])`
This method must be called first to initialize the library.

Parameter   | Type  | Description
:------|:------|:-----------
`serverUrl` | String | The URL of the API server.
`apiKey` | String | The API Key for your Absio Development Account ([TODO link](https://developer.absio.com/register))
`options` | Object [optional] | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`cacheLocal` | boolean | `true` | Set false to prevent caching information in local database and OFS.
`defaultAccess` | Array of [accessInformation](#accessInformation) | `[]` | This defines the default access for all methods that grant access to objects.
##### accessInformation
```javascript
{
    expiration: <null or Date()>,
    permissions: <"read", "read-write", "write">,
    userId: 'userIdOfUserWithDefaultAccess'
}
```

---

### `logIn(userId, password, answer[, options])`
Decrypts the key file containing the user's private keys with the provided password.  If the decryption succeeds, then a private key will be used to authenticate with the server.  The answer is used to download the key file.

Returns a Promise.

Throws an Error if the credentials are incorrect, or in the case that a connection is unavailable. Server authentication will be attempted again automatically when any method is called requiring a connection.

Parameter   | Type  | Description
:-----------|:------|:-----------
`userId` | String | The userId value is retuned at registration.  Call `register()` or use our [user creation interface](TODO place url here).
`password` | String | The password used to decrypt the key file.
`answer` | String | The answer used to reset the password or retrieve the key file from the server.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`cacheFileLocal` | boolean | `true` | Set false to prevent caching the encrypted key file in the local OFS.

---

### `create(content[, options])` -> `'containerId'`

Creates an encrypted container with the provided `content`. The container will be uploaded and access will be granted to the specified users, unless the `localAccessOnly` option is set to `true`.

Returns a Promise that resolves to the new container's ID.

Throws an Error if the connection is unavailable or an access userId is not found.

Parameter   | Type  | Description
:-----------|:------|:-----------
`content` | Buffer | Node.js Buffer for the data to be stored in the container.
`options` | Object [optional] | See table below..

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`access` | Array of user IDs (String) or [accessInformation](#accessInformation) for setting permissions and expiration | `[]`, if not defined in initialize options | The access granted to the container on upload.
`header` | Object | `{}` | Use this to store any metadata about the content.  This data is independently encrypted and can be retrieved prior to downloading and decrypting the full content.
`localAccessOnly` | boolean | `false` | This prevents uploading the container.  The container will only be accessible locally.
`type` | String | TODO define `'default type'` | A string used to categorize the container on the server.

---

### `update(id[, options])`
Updates the container with the specified ID. At least one optional parameters must be provided for an update to occur.

Returns a Promise.

Throws an Error if the connection is unavailable or an access userId is not found.

Parameter   | Type  | Description
:------|:------|:-----------
`id` | String | The ID of the container to update
`options` | Object [optional] | See table below..

Option | Type  | Default | Description
:------|:------|:--------|:-----------
|||TODO Add all rows from `create` when finalized
`content` | Buffer | null | The content to update.

---

### `getContainer(id[, options])` -> [container](#container)
Gets the secured container and decrypts it for usage. By default it downloads any required data, includes the content, and caches downloaded data locally.  See the options for overriding this behavior.

Returns a Promise that resolves to a [container](#container)

Throws an Error if the container or connection is unavailable.

Parameter   | Type  | Description
:------|:------|:-----------
`id` | String | The ID of the container to update
`options` | Object [optional] | See table below..

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`cacheLocal` | boolean | `true` | Set false to prevent caching information in local database and OFS
`includeContent` | boolean | `true` | Set to `false` to prevent downloading and decrypting content.  This is helpful when the content is very large.
`localAccessOnly` | boolean | `false` | Prevents downloading container from the server. Only locally cached containers will be available.

##### container
``` javascript
{
    access: [
        {
            expiration: <null or Date()>,
            permissions: <"read", "read-write", "write">,
            userId: 'userIdWithAccess'
        },
        ...
    ],
    content: Buffer(),
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

### `getLatestContainers([options])` -> `[{ container }]`
Downloads and decrypts any new or updated containers of the specified type. This will return all new containers since the last call of this method, unless specified in `options`.

Returns a Promise that resolves to an Array of [container](#container).

Throws an Error if the connection is unavailable.

Parameter   | Type  | Description
:------|:------|:-----------
`options` | Object [optional] | See table below..

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`startingEventId` | Number | `-1` | 0 will start from the beginning and download all containers for the current user.  Use the `storageInformation.latestEventId` field of the [container](#container) to start from existing successful event. -1 will download all new since last call.
`type` | String | TODO define `'default type'` | A string used to categorize the container on the server.
`updatesOnly` | boolean | `false` | Both new and updated data is included in the results by default. Set to `true` to only return updated containers.

---

### `delete(id[, options])`
Deletes the container from the server and local file system, unless specified in options.

Returns a Promise.

Parameter   | Type  | Description
:------|:------|:-----------
`id` | String | The ID of the container to delete
`options` | Object [optional] | See table below..

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`localAccessOnly` | boolean | `false` | Prevents deleting container from the server. Only locally cached containers will be deleted.

---

### `hash(seed)` -> `'hashedString'`
Produces a sha256 hash of the specified seed.

Returns a string with the hashed value.

Parameter   | Type  | Description
:------|:------|:-----------
`seed` | String | Seed for producing the hash
