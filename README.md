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
The `userId`, `password`, and `answer` used below are the credentials for two existing users.  To simplify the example the users are called Alice and Bob. [Users](#users) can be created with the `register()` method or with our web-based secure user creation utility. For more details see the [Users](#users) section above.

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
   await securedContainer.logIn(alicesId, alicesPassword, alicesAnswer);
   ```
4. Start creating secured containers:

   ``` javascript
   const sensitiveData = new Buffer('information to protect');
   const containerAccess = [{
       userId: bobsId,
       permission: 'read-write',
       expiration: new Date(2020)
   }];

   const containerId = await securedContainer.create(sensitiveData, { access: containerAccess });
   ```

5. Securely access these containers from another system:

   ``` javascript
   await securedContainer.logIn(bobsId, bobsPassword, bobsAnswer);
   const latestContainers = await securedContainer.getLatest();

   // Also can use a known container ID returned from create.
   const container = await securedContainer.get(knownContainerId);
   ```

## Usage
The following usage examples requires that the general setup in [Getting Started](#getting-started) has been completed.

#### Create
This is an example of creating a container that contains sensitive data.

``` javascript
const secretData = new Buffer('Secret Report...000-00-0000...');

// Optional: Define a custom header that is bound to the content with encryption.
const containerHeader = {
    recordCount: 500,
    applicationEnforceableMetadata: {
        allowPrint: false,
        allowExport: false
    }
};

// Optional: Grant access to the container with permissions and/or expiration.
const containerAccess = [{
    {
        userId: trustedSystemId,
        expiration: new Date(2022)
        permissions: {
            read: true,
            write: false,
            modifyAccess: false
        }
    }
];

// Create the container
const containerId = await securedContainer.create(secretData, {
    access: containerAccess
    header: containerHeader
});
```

#### Get

This demonstrates the ways to get containers.

``` javascript
// Get the content with a known ID
const container = await securedContainer.get(knownContainerId);

// The application expects to be granted access to one or more containers.
const latestContainers = await securedContainer.getLatest();

// This can be narrowed further to return only the latest of a given type.
const latestOfType = await securedContainer.getLatest({ type: 'exampleType' });
```


#### Update

The container [created above](#create) needs to be updated later with additional data containing sensitive information.  Also a new trusted system is given full access to this report.

``` javascript
// Get the container securely
const container = await securedContainer.get(containerId);

// Do custom business logic to update the content and header.  This can be anything.
updateSecretData(container.content, recordsToAdd);
container.header.recordCount += recordsToAdd.length;

// Grant additional access with full permissions and no expiration
container.access.push({
    userId: addedSystemId,
    permissions: {
        read: true,
        write: true,
        modifyAccess: true
    }
});

// Update the container
await securedContainer.update(container);
```

#### Delete

The container with secret data [created above](#create) should no longer be accessible.

``` javascript
await securedContainer.delete(containerId);
```

### Possible 418 Intelligence Usage
Below are three examples specific to our understanding of the simplest usage in the 418 use cases.  Some of the Promises could be executed in parallel for maximum efficiency, but to simplify the examples this is not included below.

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
    const unstructuredDataContainers = await securedContainer.getLatest({ type: 'unstructured-data' });

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
    const netFlowContainers = await securedContainer.getLatest({ type: 'net-flow-data' });

    for (let container of netFlowContainers) {
        const report = performAnalysis(container.content);

        await securedContainer.create(report, {
            access: [trustedDataBrokerID],
            type: 'report'
        });
    }
}

async function processUpdatedReports() {
    const updatedReportContainers = await securedContainer.getLatest({
        type: 'report'
        updatesOnly: true
    });

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
`apiKey` | String | The API Key is required for using the API server.
`options` | Object [optional] | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`applicationName` | String | `''` | The API server uses the application name to identify different applications.  Obfuscated File System data will be saved separately from other applications when this is specified.
`cacheLocal` | boolean | `true` | By default we cache information in a local database and Obfuscated File System.  This allows for offline access and greater efficiency.  Set to `false` to prevent this behavior.
`obfuscatedFileSystemSeed` | String | `apiKey` | By default we use the specified `apiKey` as the seed to the Obfuscated File System.  If would like greater granularity and separation of data, then provide a unique static string for the seed value.
`rootDirectory` | String | `'./'` | By default the root directory of the database and Obfuscated File System will be the current directory.  

---

### `register(password, question, answer)` -> `'userId'`
**Important:** The `password` and `answer` should be kept secret.  We recommend using long and complex values with numbers and/or symbols.  Do not store them publicly in plain text.

Generates private keys and registers a new user on the API server.  This user's private keys are encrypted with the `password` to produce a key file.  The `answer` is used to reset the password and download the key file. Our web-based user creation utility can also be used to securely generate static users.

Returns a Promise that resolves to the new user's ID.

Throws an Error if the connection is unavailable.

Parameter   | Type  | Description
:-----------|:------|:-----------
`password` | String | The password used to encrypt the key file.
`question` | String | The question should only be used as a hint to remember the answer. This string is stored in plain text and should not contain sensitive information.
`answer` | String | The answer used to reset the password or retrieve the key file from the server.

---

### `logIn(userId, password, answer[, options])`
Decrypts the key file containing the user's private keys with the provided password.  If the decryption succeeds, then a private key will be used to authenticate with the server.  If the key file is not cached locally, the answer is used to download the key file.

Returns a Promise.

Throws an Error if the credentials are incorrect, or in the case that a connection is unavailable. Server authentication will be attempted again automatically when any method is called requiring a connection.

Parameter   | Type  | Description
:-----------|:------|:-----------
`userId` | String | The userId value is returned at registration.  Call `register()` or use our [user creation interface](TODO place url here).
`password` | String | The password used to decrypt the key file.
`answer` | String | The answer used to reset the password or retrieve the key file from the server.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`cacheFileLocal` | boolean | `true` | By default we cache the encrypted key file in the local Obfuscated File System for offline and greater efficiency.  Set to `false` to prevent this from being cached.

---

### `create(content[, options])` -> `'containerId'`

Creates an encrypted container with the provided `content`. The container will be uploaded and access will be granted to the specified users, unless the `localAccessOnly` option is set to `true`.  

The container will never expire for the owner.  The owner is automatically granted full permission to the container.

Returns a Promise that resolves to the new container's ID.

Throws an Error if the connection is unavailable or an access userId is not found.

Parameter   | Type  | Description
:-----------|:------|:-----------
`content` | [Buffer](https://nodejs.org/dist/latest-v6.x/docs/api/buffer.html) | Node.js [Buffer](https://nodejs.org/dist/latest-v6.x/docs/api/buffer.html) for the data to be stored in the container.
`options` | Object [optional] | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`access` | Array of user IDs (String) or [accessInformation](#accessInformation) for setting permissions and expiration | `[]`, if not defined in initialize options | The access granted to the container on upload.
`header` | Object | `{}` | Use this to store any metadata about the content.  This data is independently encrypted and can be retrieved prior to downloading and decrypting the full content.
`uploadToServer` | boolean | `true` | By default we upload the secured container to the server for backup and granting access.  Set to `false` to prevent server storage of encrypted containers.
`type` | String | `null` | A string used to categorize the container on the server.

#### accessInformation
```javascript
{
    expiration: <null or Date()>,
    permissions: { // Default permissions:
        read: true,
        write: false,
        modifyAccess: false
    },
    userId: 'userIdOfUserWithAccess'
}
```

---

### `update(container[, options])`
TODO based on confluence feedback

---

### `update(id[, options])`
Updates the container with the specified ID. At least one optional parameters must be provided for an update to occur.

Returns a Promise.

Throws an Error if the connection is unavailable or an access userId is not found.

Parameter   | Type  | Description
:------|:------|:-----------
`id` | String | The ID of the container to update
`options` | Object [optional] | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
|||TODO Add all rows from `create` when finalized
`content` | [Buffer](https://nodejs.org/dist/latest-v6.x/docs/api/buffer.html) | null | The content to update.

---

### `get(id[, options])` -> [container](#container)
Gets the secured container and decrypts it for usage. By default it downloads any required data, includes the content, and caches downloaded data locally.  Only encrypted data is cached locally and it is stored in an Obfuscated File System.  See the options for overriding this behavior.

Returns a Promise that resolves to a [container](#container)

Throws an Error if the container or connection is unavailable.

Parameter   | Type  | Description
:------|:------|:-----------
`id` | String | The ID of the container to update
`options` | Object [optional] | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`cacheLocal` | boolean | `true` | By default we cache information in a local database and Obfuscated File System.  This allows for offline access and greater efficiency.  Set to `false` to prevent this behavior.
`includeContent` | boolean | `true` | Set to `false` to prevent downloading and decrypting content.  This is helpful when the content is very large.
`localAccessOnly` | boolean | `false` | Prevents downloading container from the server. Only containers cached locally in the Obfuscated File System will be available.

#### container
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

### `getLatest([options])` -> [`[ { container } ]`](#container)
Downloads and decrypts any new or updated containers. This will return all new containers since the last call of this method, unless specified in `options`. Options can be used to change the criteria for the containers returned by this method.

Returns a Promise that resolves to an Array of [containers](#container).

Throws an Error if the connection is unavailable.

Parameter   | Type  | Description
:------|:------|:-----------
`options` | Object [optional] | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`startingEventId` | Number | `-1` | 0 will start from the beginning and download all containers for the current user.  Use the `storageInformation.latestEventId` field of the [container](#container) to start from existing successful event. -1 will download all new since last call.
`type` | String | TODO define `'default type'` | Only containers of the specified type will be downloaded. Type is a string used to categorize containers on the server.
`updatesOnly` | boolean | `false` | Both new and updated data is included in the results by default. Set to `true` to only return updated containers.

---

### `delete(id[, options])`
Deletes the container from the server and revokes access for all users, unless specified in options. Any data relating to this container will be deleted from the local Obfuscated File System.

Returns a Promise.

Parameter   | Type  | Description
:------|:------|:-----------
`id` | String | The ID of the container to delete
`options` | Object [optional] | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`localAccessOnly` | boolean | `false` | Prevents deleting container from the server. Only locally cached containers will be deleted.

---

### `getAccessNotifications(id)` -> [`[ { accessNotification } ]`](#accessNotification)
TODO

#### accessNotification
TODO

---

`getSecurityQuestion(userId)` -> `'security question for the current user'`
TODO

---

`resetPassword(currentAnswer, newPassword)`
TODO

---

`changePassword(currentAnswer, currentPassword, newPassword)`
TODO

---

`changeQuestionAndAnswer(currentAnswer, currentPassword, newQuestion, newAnswer)`
TODO

---

### `hash(seed)` -> `'hashedString'`
Produces a sha256 hash of the specified seed.

Returns a string with the hashed value.

Parameter   | Type  | Description
:------|:------|:-----------
`seed` | String | Seed for producing the hash
