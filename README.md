# Absio Secured Container

Protect your application's sensitive data with Absio's Secured Containers.

## Index

* [Overview](#overview)
* [Getting Started](#getting-started)
* [Usage](#usage)
* [API](#api)

## Overview
We use AES256 [encryption](#encryption) with unique keys for each Absio Secured Container to protect your application's data.  For offline access and efficiency the Secured Containers are stored in Absio's [Obfuscated File System](#obfuscated-file-system).

### Asynchronous
* All Secured Container functions return a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) and execute asynchronously, except for [`initialize()`](#initializeserverurl-apikey-options).
  * Use the existing [`.then()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then) and [`.catch()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/catch) methods to handle the promises.
  * Execute multiple asynchronous Secured Container methods in parallel using [`Promise.all()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all) or [`Promise.race()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/race).
* If this prevents you from using Absio Secured Container in your application, then please contact us.
* Optionally use the async-await syntax enabled by [Babel](https://babeljs.io/) as shown in the [Usage](#usage) section.
* For greater browser support we suggest using the [ES6 Promise](https://github.com/stefanpenner/es6-promise) polyfill.

### Users
* A user is an entity that has its own set of private keys.  
* Create users with the [`register()`](#registerpassword-question-backuppassphrase---userid) function.
* A user's public keys are registered with an Absio Server under a static User ID.
  * Public keys are publicly available to other users for granting access to containers.
  * Public keys are used by the server to validate user actions.
* Each user can [create](#createcontent-options---containerid) Secured Containers that are uniquely [encrypted](#encryption).
* Optionally a user can grant [access](#accessinformation) to a container for sharing with unique permissions or expiration to another set of users.

### Key File
* A [user's](#users) Key File is an AES256 [encrypted](#encryption) file containing private keys and password reset.  
* A [user's](#users) password is used to encrypt their private keys.  This mechanism allows Absio Secured Containers to be accessible offline.
* A backup passphrase can be provided to synchronize a Key File between devices and enable a secure password reset.

### Obfuscated File System
* All Secured Containers and [Key Files](#key-file) can be securely in Absio's Obfuscated File System.  
* This module automatically obfuscates the names and randomizes folder structure of the encrypted files.  
* Increases security by making attacks on the individually [encrypted](#encryption) containers more difficult.
* The seed for obfuscation combines user, application, and server information to partition the data.

### Encryption
* A user's private keys are stored in [Key File](#key-file) encrypted with AES256 using a key derived from the user's password.
  * A [Key File](#key-file) contains both signing and derivation private keys.
  * The encryption key is derived using the Password Based Key Derivation Function 2 (PBKDF2).
* Every Absio Secured Container has a unique set of secret keys.
  * Secret Keys are stored encrypted and used to securely access the container.
  * HMAC-SHA256 keys are used for content validation to mitigate content tampering.
  * AES256 keys are used to individually encrypt the header and content of the Secured Container.
* The secret keys for an Absio Secured Container are uniquely encrypted for each user that can access the container.
  * This encryption of the secret keys uses Static-ephemeral Diffie-Hellman Key Exchange (DHKE) based upon a user's public derivation key.
  * This process encrypts the secret keys such that they can only be decrypted with the user's corresponding private derivation key.
  * Encrypted container keys are signed with the creator's private keys to mitigate Man-in-the-Middle attacks.


## Getting Started
The `userId`, `password`, and `backupPassphrase` used below are the credentials for two existing users.  To simplify the example the users are called Alice and Bob. [Users](#users) can be created with the `register()` method or with our web-based secure user creation utility. For more details see the [Users](#users) section above.

1. Installation:

   ```
   npm install absio-secured-container
   ```

2. Import as regular module and initialize:

   ``` javascript
   var securedContainer = require('absio-secured-container');
   ```
3. Initialize the module and log in with an account:  

   ``` javascript
   securedContainer.initialize('your.absioApiServer.com', yourApiKey);
   await securedContainer.logIn(alicesId, alicesPassword, alicesBackupPassphrase);
   ```
4. Start creating secured containers:

   ``` javascript
   const sensitiveData = new Buffer('Sensitive Data...000-00-0000...');
   const containerAccess = [{
       userId: bobsId,
       permission: 'read-write',
       expiration: new Date(2020)
   }];

   const containerId = await securedContainer.create(sensitiveData, { access: containerAccess });
   ```

5. Securely access these containers from another system:

   ``` javascript
   await securedContainer.logIn(bobsId, bobsPassword, bobsBackupPassphrase);
   const latestContainers = await securedContainer.getLastestEvents();

   // Also can use a known container ID returned from create.
   const container = await securedContainer.get(knownContainerId);
   ```

## Usage
The following usage examples requires that the general setup in [Getting Started](#getting-started) has been completed.

#### Create
This is an example of creating a container that contains sensitive data.

``` javascript
const sensitiveData = new Buffer('Sensitive Data...000-00-0000...');

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
    userId: trustedSystemId,
    expiration: new Date(2022)
    permissions: {
        read: true,
        write: false,
        modifyAccess: false
    }
}];

// Create the container
const containerId = await securedContainer.create(sensitiveData, {
    access: containerAccess
    header: containerHeader
});
```

#### Get

This demonstrates the ways to get containers.

``` javascript
// Get the content with a known ID
const container = await securedContainer.get(knownContainerId);

// The API Server creates an event for when a container is new, updated, deleted, or first accessed.
const latestEvents = await securedContainer.getLastestEvents();

// Events contain the corresponding container ID for getting the related container.
const eventContainerId = latestEvents[0].containerId;
const containerFromEvent = await securedContainer.get(eventContainerId);

// Optionally can be filtered to return only the latest events of a given container type and/or event action.
const latestUpdatedOfType = await securedContainer.getLastestEvents({
    containerType: 'exampleType',
    eventAction: 'updated'
});
```


#### Update

The container's content and header [created above](#create) needs to be updated later with additional sensitive information.  Also a new trusted system is given full access to this report.

``` javascript
// Get the container securely.
const container = await securedContainer.get(containerId);

// Make any needed update the content or header directly.
updateContent(container.content);
updateHeader(container.header);

// Can add access with full permissions and no expiration.
container.access.push({
    userId: addedSystemId,
    permissions: {
        read: true,
        write: true,
        modifyAccess: true,
        viewAccess: true
    }
});

// Update the container.
await securedContainer.update(container);
```

Update can also be used to redefine any field by referencing the container ID with any updated fields.  This shows redefining the type of a container with known ID.

``` javascript
await securedContainer.update(knownContainerId, { type: 'redefinedContainerType' });
```

#### Delete

The container with secret data [created above](#create) should no longer be accessible.

``` javascript
await securedContainer.deleteContainer(containerId);
```

### Possible 418 Intelligence Usage
Below are three examples specific to our understanding of the simplest usage in the 418 use cases.  Some of the Promises could be executed in parallel for maximum efficiency, but to simplify the examples this is not included below.

#### Customer System

Encrypt unstructured data and share with the Trusted Data Broker.

``` javascript
async function shareUnstructuredData(unstructuredData) {
    const containerOptions = {
        access: [trustedDataBrokerId],
        type: 'unstructured-data'
    };

    await securedContainer.create(unstructuredData, containerOptions);
}
```

Get report container events and corresponding secured containers. Then update the reports based upon validity.

``` javascript

async function updateReportsForValidity() {
    const reportContainerEvents = await securedContainer.getLastestEvents({ containerType: 'report' });

    for (let reportEvent of reportContainerEvents) {
        const reportContainer = await securedContainer.get(reportEvent.containerId);

        reportContainer.content = processReportForValidity(reportContainer.content);
        await securedContainer.update(reportContainer);
    }
}

```

#### Trusted Data Broker

Get unstructured data and convert into NetFlow format and obfuscated data map.  Encrypt and Share the NetFlow data with the [Analysis System](#analysis-system) and the obfuscated data map with the [Customer System](#customer-system).

``` javascript
async function processUnstructuredData() {
    const unstructuredDataEvents = await securedContainer.getLastestEvents({ containerType: 'unstructured-data' });

    for (let containerEvent of unstructuredDataEvents) {
        const container = securedContainer.get(containerEvent.containerId);
        const results = createNetFlowDataWithDataMap(container.content);

        await shareNetFlowData(results.netFlowFormat);
        await shareObfuscatedDataMap(results.obfuscatedDataMap);
    }
}

async function shareNetFlowData(netFlowData) {
    const containerOptions = {
        access: [analysisSystemId],
        containerType: 'net-flow-data'
    };

    await securedContainer.create(netFlowData, containerOptions);
}

async function shareObfuscatedDataMap(obfuscatedDataMap) {
    const containerOptions = {
        access: [customerSystemId],
        type: 'obfuscated-data-map'
    };

    await securedContainer.create(obfuscatedDataMap, containerOptions);
}
```

Get added report container events and redefine the access and corresponding permissions. The redefined permissions ensure that the [Customer System](#customer-system) and [Analysis System](#analysis-system) cannot know about each other, while still maintaining read and write access to the report.

``` javascript
const limitedPermissions = {
    read: true,
    write: true,
    modifyAccess: false,
    viewAccess: false
}

const redefinedAccess = [
    {
        userId: analysisSystemId,
        permissions: {
            read: true,
            write: true,
            modifyAccess: true,
            viewAccess: true
        }    
    },
    {
        userId: customerSystemId,
        permissions: limitedPermissions
    },
    {
        userId: analysisSystemId,
        permissions: limitedPermissions
    }
]

async function redefineReportAccess() {
    const reportContainerEvents = await securedContainer.getLatestEvents({
        eventAction: 'added'
        containerType: 'report',
    });

    for (let reportEvent of reportContainerEvents) {
        await securedContainer.update(reportEvent.containerId, {
            access: redefinedAccess
        });
    }
}
```

#### Analysis System

Get the containers with NetFlow formatted data.  After performing analysis on the data, create a Secured Container that the [Trusted Data Broker](#trusted-data-broker) can access with full permissions.

``` javascript
async function processNetFlowData() {
    const netFlowContainerEvents = await securedContainer.getLastestEvents({ containerType: 'net-flow-data' });

    const reportContainerAccess = [{
        userId: trustedDataBrokerId,
        permissions: {
            read: true,
            write: true,
            modifyAccess: true,
            viewAccess: true
        }
    }];

    for (let containerEvent of netFlowContainerEvents) {
        const container = await securedContainer.get(containerEvent.containerId);
        const report = performAnalysis(container.content);

        await securedContainer.create(report, {
            access: reportContainerAccess,
            containerType: 'report'
        });
    }
}
```

Get containers with reports updated by the [Customer System](#customer-system).  Use updated report to adjust the algorithms of the Analysis System.

``` javascript
async function processUpdatedReports() {
    const updatedReportEvents = await securedContainer.getLastestEvents({
        containerType: 'report'
        eventAction: 'updated'
    });

    for (let reportEvent of updatedReportEvents) {
        const reportContainer = await securedContainer.get(reportEvent.containerId);
        updateSystemForReport(reportContainer.content);
    }
}
```

## API
* Container
  * [create(content[, options])](#createcontent-options---containerid)
  * [deleteContainer(id[, options])](#deleteid-options)
  * [get(id[, options])](#getid-options---container)
    * [container](#container)
  * [getLastestEvents([options])](#getlatesteventsoptions-----container--)
  * [getAccessNotifications(id)](#getaccessnotificationsid-----accessnotification--)
    * [accessNotification](#accessnotification)
  * [update(container[, options])](#updatecontainer-options)
  * [update(id[, options])](#updateid-options)
* General
  * [hash(seed) ](#hashseed---hashedstring)
  * [initialize(serverUrl, apiKey[, options])](#initializeserverurl-apikey-options)
* User Accounts
  * [changeBackupCredentials(currentPassphrase, currentPassword, newReminder, newPassphrase)](#changebackupcredentialscurrentpassphrase-currentpassword-newreminder-newpassphrase)
  * [changePassword(currentPassphrase, currentPassword, newPassword)](#changepasswordcurrentpassphrase-currentpassword-newpassword)
  * [getBackupReminder(userId)](#getbackupreminderuserid---reminder-for-the-backup-passphrase)
  * [logIn(userId, password, backupPassphrase[, options])](#loginuserid-password-backuppassphrase-options)
  * [register(password, question, backupPassphrase)](#registerpassword-question-backuppassphrase---userid)
  * [resetPassword(userId, backupPassphrase, newPassword)](#resetpassworduserid-backuppassphrase-newpassword)

### `initialize(serverUrl, apiKey[, options])`

This method must be called first to initialize the module.

Parameter   | Type  | Description
:------|:------|:-----------
`serverUrl` | String | The URL of the API server.
`apiKey` | String | The API Key is required for using the API server.  Contact Absio to obtain an API Key.
`options` | Object [optional] | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`applicationName` | String | `''` | The API server uses the application name to identify different applications.
`cacheLocal` | boolean | `true` | By default we cache information in a local database and Obfuscated File System.  This allows for offline access and greater efficiency.  Set to `false` to prevent this behavior.
`obfuscatedFileSystemSeed` | String | `apiKey` | By default the specified `apiKey` and user information are used as the seed to the Obfuscated File System.  If would like greater granularity and separation of data, then provide a unique static string for the seed value.
`rootDirectory` | String | `'./'` | By default the root directory of the Obfuscated File System will be the current directory.  The database and encrypted data is stored inside the Obfuscated File System.

---

### `register(password, reminder, backupPassphrase)` -> `'userId'`

**Important:** The `password` and `backupPassphrase` should be kept secret.  We recommend using long and complex values with numbers and/or symbols.  Do not store them publicly in plain text.

Generates private keys and registers a new user on the API server.  This user's private keys are encrypted with the `password` to produce a key file.  The `backupPassphrase` is used to reset the password and download the key file. Our web-based user creation utility can also be used to securely generate static users.

Returns a Promise that resolves to the new user's ID.

Throws an Error if the connection is unavailable.

Parameter   | Type  | Description
:-----------|:------|:-----------
`password` | String | The password used to encrypt the key file.
`reminder` | String | The reminder should only be used as a hint to remember the `backupPassphrase`. This string is stored in plain text and should not contain sensitive information.
`backupPassphrase` | String | The `backupPassphrase` can be used later to reset the password or to allow logging in from another system.

---

### `logIn(userId, password, backupPassphrase[, options])`

Decrypts the key file containing the user's private keys with the provided password.  If the decryption succeeds, then a private key will be used to authenticate with the server.  If the key file is not cached locally, the `backupPassphrase` is used to download the key file.

Returns a Promise.

Throws an Error if the `password` or `backupPassphrase` are incorrect, the `userId` is not found, or a connection is unavailable. In the last case the server authentication will be attempted again automatically when any method is called requiring a connection.

Parameter   | Type  | Description
:-----------|:------|:-----------
`userId` | String | The userId value is returned at registration.  Call `register()` or use our [user creation interface](TODO place url here).
`password` | String | The password used to decrypt the key file.
`backupPassphrase` | String | The `backupPassphrase` is used retrieve the key file from the server.

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
`access` | Array of user IDs (String) or [accessInformation](#accessInformation) for setting permissions and expiration | `[]` | Access will be granted to the users in this Array with any specified permissions or expiration.
`header` | Object | `{}` | This will be serialized with `JSON.Stringify()`, independently encrypted, and can be retrieved prior to downloading and decrypting the full content. Use this to store any data related the content.
`uploadToServer` | boolean | `true` | By default we upload the secured container to the server for backup and granting access.  Set to `false` to prevent server storage of encrypted containers.
`type` | String | `null` | A string used to categorize the container on the server.

#### accessInformation
```javascript
{
    expiration: <null or Date()>,
    permissions: { // These are default permissions, if none are specified.
        read: true,
        write: false,
        modifyAccess: false,
        viewAccess: true
    },
    userId: 'userIdGrantedAccess'
}
```

---

### `update(container[, options])`

Updates the changed fields of the specified [container](#container). Both the local and server copies of the secured container will be updated, unless specified in the `uploadToServer` option.

Returns a Promise.

Throws an Error if the connection is unavailable or an ID is not found.

Parameter   | Type  | Description
:-----------|:------|:-----------
[`container`](#container) | Object | The container object that has been updated after being returned from `get()`.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`uploadToServer` | boolean | `true` | By default we upload the secured container to the server for backup and granting access.  Set to `false` to prevent server storage of encrypted containers.

---

### `update(id[, options])`

Updates the container with the specified ID. At least one optional parameters must be provided for an update to occur.

Returns a Promise.

Throws an Error if the connection is unavailable or an ID is not found.

Parameter   | Type  | Description
:------|:------|:-----------
`id` | String | The ID of the container to update
`options` | Object [optional] | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`access` | Array of user IDs (String) or [accessInformation](#accessInformation) for setting permissions and expiration | `null` | The access granted to the container on upload. If not specified the currently defined access will be left unchanged.
`content` | [Buffer](https://nodejs.org/dist/latest-v6.x/docs/api/buffer.html) | null | The content to update.
`header` | Object | `null` | This will be serialized with `JSON.Stringify()`, independently encrypted, and can be retrieved prior to downloading and decrypting the full content. Use this to store any data related the content.
`uploadToServer` | boolean | `true` | By default we upload the secured container to the server for backup and granting access.  Set to `false` to prevent server storage of encrypted containers.
`type` | String | `null` | A string used to categorize the container on the server.

---

### `deleteContainer(id[, options])`

Deletes the container from the server and revokes access for all users, unless specified in options. Any data relating to this container will be deleted from the local Obfuscated File System.

Returns a Promise.

Throws an Error if the ID is not found or a connection is unavailable.  If the `localAccessOnly` option is set to `true`, then a connection is not needed.

Parameter   | Type  | Description
:------|:------|:-----------
`id` | String | The ID of the container to delete.
`options` | Object [optional] | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`localAccessOnly` | boolean | `false` | Prevents deleting container from the server. Only locally cached containers will be deleted.

---

### `get(id[, options])` -> [container](#container)

Gets the secured container and decrypts it for usage. By default it downloads any required data, includes the content, and caches downloaded data locally.  Only encrypted data is cached locally and it is stored in an Obfuscated File System.  See the options for overriding this behavior.

Returns a Promise that resolves to a [container](#container)

Throws an Error if the container or connection is unavailable.  If the `localAccessOnly` option is set to `true`, then a connection is not needed.

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
            permissions: {
                read: true,
                write: false,
                modifyAccess: false,
                viewAccess: true
            },
            userId: 'userIdWithAccess'
        },
        ...
    ],
    content: Buffer(),
    created: Date(),
    header: {},
    id: 'IdAsGuid',
    keys: {},
    latestEventId: 543,
    modified: Date(),
    modifiedBy: 'userId that modified',
    owner: 'userId of owner',
    securedLength: 12345,
    type: 'containerType'
}
```

---

### `getLastestEvents([options])` -> [`[ { containerEvent } ]`](#containerevent)

Gets all new container events since the last call of this method, unless specified with `startingEventId` in `options`. Options can be used to change the criteria for the container events returned by this method.

Returns a Promise that resolves to an Array of [container events](#containerevent).

Throws an Error if the connection is unavailable.

Parameter   | Type  | Description
:------|:------|:-----------
`options` | Object [optional] | See table below.

Option | Type  | Default | Description
:------|:------|:--------|:-----------
`startingEventId` | Number | `-1` | 0 will start from the beginning and download all containers for the current user.  Use the `storageInformation.latestEventId` field of the [container](#container) to start from existing successful event. -1 will download all new since last call.
`containerType` | String | `null` | Only containers of the specified type will be downloaded. Type is a string used to categorize containers on the server.
`eventAction` | String | `'all'` | All container actions are included in the results by default. Possible values are: `'all'`, `'accessed'`, `'added'`, `'deleted'`, or `'updated'`

#### containerEvent

``` javascript
{
  action: "Always one of 'accessed', 'added', 'deleted', or 'updated'.",
  changes : "Information about what has changed. For example: { fieldThatChanged: 'updatedValue' }",
  appName: "The name of the application responsible for the action, optionally specified in the initialize() method.",
  date: "ISO formatted date string corresponding to when the event occurred.",
  eventId: "An integer ID value for the event.",
  containerId: "The ID for container that this event relates to.",
  lastModified: "ISO formatted date string corresponding to when the the container was last modified.",
  containerType: "The type of the container as specified upon creation or last update.",
  relatedUserId: "If this event relates to another user, this field will be set to that user's GUID.  Currently only applies to 'accessed' actions.",
}
```

---

### `getAccessNotifications(id)` -> [`[ { accessNotification } ]`](#accessnotification)

Gets the dates that users first accessed the secured container on the server.

Returns a promise that resolves to an Array of [accessNotification](#accessnotification).

Throws an Error if the ID is not found or a connection is unavailable.

Parameter   | Type  | Description
:------|:------|:-----------
`id` | String | The ID of the container.

#### accessNotification
``` javascript
{
    userId: 'userIdThatAccessed'
    firstAccessed: Date() // Date first accessed by corresponding userId
}
```

---

### `getBackupReminder(userId)` -> `'Reminder for the backup passphrase'`
Gets the publicly accessible reminder for the user's backup passphrase.  If no ID is provided, the user ID for the currently logged in user will be used.

Returns a Promise that resolves to the user's reminder.

Throws an exception if the connection is unavailable.

Parameter   | Type  | Description
:-----------|:------|:-----------
`userId` | String | A string ID representing the user.

---

### `resetPassword(userId, backupPassphrase, newPassword)`
If user's password is forgotten, then use this to reset a user's password. Call `getBackupReminder(userId)` to get the reminder for the `backupPassphrase`.

Returns a Promise.

Throws an Error if the `backupPassphrase` is incorrect or user ID is not found.  If the Key File is not cached locally, then an Error will be thrown if the connection is unavailable.

Parameter   | Type  | Description
:-----------|:------|:-----------
`userId` | String | A string ID representing the user.
`backupPassphrase` | String | The `backupPassphrase` is set up during registration of the account.  This is used to reset the password.
`newPassword` | String | The new password for the user.

---

### `changePassword(currentPassphrase, currentPassword, newPassword)`
Change the password for the current user.  The current `backupPassphrase` is required for updating the backup.

Returns a Promise.

Throws an Error if the `currentPassphrase` or `currentPassword` is incorrect.

Parameter   | Type  | Description
:-----------|:------|:-----------
`currentPassphrase` | String | The `currentPassphrase` is set up during registration of the account.  This is used to reset the password.
`currentPassword` | String | The current password for the user.
`newPassword` | String | The new password for the user.

---

### `changeBackupCredentials(currentPassphrase, currentPassword, newReminder, newPassphrase)`
Change the backup credentials for the account.  Use a secure value for the passphrase as it can be used to reset the user's password.

Returns a Promise.

Throws an Error if the `currentPassphrase` or `currentPassword` is incorrect.  

Parameter   | Type  | Description
:-----------|:------|:-----------
`currentPassphrase` | String | The `currentPassphrase` setup during registration of the account.
`currentPassword` | String | The current password for the user.
`newReminder` | String | The new backup reminder for the user's passphrase.  The reminder is publicly available in plain text.  Do not include sensitive information or wording that allows the passphrase to be easily compromised.
`newPassphrase` | String | The new backup passphrase for the user.  Use a secure value for this.  This can be used to reset the password for the user's account.

---

### `hash(seed)` -> `'hashedString'`
Produces a sha256 hash of the specified seed.

Returns a string with the hashed value.

Parameter   | Type  | Description
:------|:------|:-----------
`seed` | String | Seed for producing the hash
