# Storage Buckets


## Authors:

* Ayu Ishii
* Victor Costan


## Participate

* [Issue tracker](https://github.com/pwnall/storage-buckets/issues)
* [Issue on the Storage specification](https://github.com/whatwg/storage/issues/2)


## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Use Cases](#use-cases)
- [Setting up buckets](#setting-up-buckets)
- [Getting the storage policies associated with a bucket](#getting-the-storage-policies-associated-with-a-bucket)
- [Accessing storage APIs from buckets](#accessing-storage-apis-from-buckets)
- [Deleting buckets](#deleting-buckets)
- [Enumerating buckets](#enumerating-buckets)
- [Storage policy: Persistence](#storage-policy-persistence)
- [Storage policy: Durability](#storage-policy-durability)
- [Getting a bucket's quota usage](#getting-a-buckets-quota-usage)
- [Reserving quota for a bucket](#reserving-quota-for-a-bucket)
- [The default bucket](#the-default-bucket)
- [Storage buckets and service workers](#storage-buckets-and-service-workers)
- [Storage buckets and the Clear-Site-Data](#storage-buckets-and-the-clear-site-data)
- [Key Scenarios](#key-scenarios)
  - [Storage Eviction](#storage-eviction)
  - [Partitioned Storage](#partitioned-storage)
  - [Quota Management](#quota-management)
- [Detailed design discussion](#detailed-design-discussion)
  - [Bucket names](#bucket-names)
  - [Bucket titles](#bucket-titles)
  - [Storage policy naming](#storage-policy-naming)
  - [Durability guarantees](#durability-guarantees)
- [Considered alternatives](#considered-alternatives)
  - [Expose the API off of navigator.storage.buckets](#expose-the-api-off-of-navigatorstoragebuckets)
  - [Separate intents for creating a bucket and opening an existing bucket](#separate-intents-for-creating-a-bucket-and-opening-an-existing-bucket)
  - [Bucket designations for each storage API](#bucket-designations-for-each-storage-api)
  - [Allow all safe characters for HTTP headers in bucket names](#allow-all-safe-characters-for-http-headers-in-bucket-names)
  - [No length restriction for bucket names](#no-length-restriction-for-bucket-names)
  - [Relaxed length restrictions for bucket names](#relaxed-length-restrictions-for-bucket-names)
  - [Alternative name for the bucket `title` property](#alternative-name-for-the-bucket-title-property)
  - [Language maps for bucket titles](#language-maps-for-bucket-titles)
  - [Enumerate all buckets using an async iterator](#enumerate-all-buckets-using-an-async-iterator)
  - [Separate durability options for flushing to the storage device vs media](#separate-durability-options-for-flushing-to-the-storage-device-vs-media)
  - [Separate durability option for application-level buffers](#separate-durability-option-for-application-level-buffers)
  - [Default to strict durability](#default-to-strict-durability)
  - [Support changing a bucket's durability policy](#support-changing-a-buckets-durability-policy)
  - [Synchronous access to a bucket's storage policies](#synchronous-access-to-a-buckets-storage-policies)
  - [Keep deleted buckets alive while there are references to them](#keep-deleted-buckets-alive-while-there-are-references-to-them)
  - [Integrate storage buckets with DOM Storage](#integrate-storage-buckets-with-dom-storage)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


## Introduction

This explainer proposes changes to
[the Storage Standard](https://storage.spec.whatwg.org/).

The core of the proposal is granting sites the ability to create multiple
[storage buckets](https://storage.spec.whatwg.org/#buckets), where the
user agent may choose to delete each bucket independently of other buckets. By
contrast, today's user agents have a binary choice of either persisting or
deleting all the data stored by a site.

This proposal also entails designating buckets as the recommended unit for
managing existing storage policy (quota) and new storage policies, such as
expiration, persistence, and encryption. Each storage bucket can store
data associated with established storage APIs such as
[IndexedDB](https://w3c.github.io/IndexedDB/) and
[CacheStorage](https://w3c.github.io/ServiceWorker/#cachestorage), so defining
storage policies at the bucket level avoids the need of introducing mechanisms
for specifying the policies for each individual API.

## Goals

* Allow web applications to evict partitions of data.

* Allow web developers to specify eviction ordering.

* Allow web applications to easily evict service workers without clearing data
  for the entire domain.

* Allow web developers to reserve quota

* Allow web developers to express performance, durability and other trade-off
  decisions on partitions of data.

* Ensure privacy across accounts on shared devices

* Allow users to have control over which data to evict, and prevent important
  data from being deleted.

## Non-goals

[If there are “adjacent” goals which may appear to be in scope but aren’t,
enumerate them here. This section may be fleshed out as your design
progresses and you encounter necessary technical and other trade-offs.]

## Use Cases

*See [Key Scenarios](#key-scenarios) for more detail on use cases.*

- **Storage eviction**: by allowing applications to have more control over
prioritization and organization, it allows them to decide on storage
trade-offs themselves upon storage eviction during low disk space instead of
losing all data.

- **Storage partitioning**: allowing applications to group data. Applications
can also choose to group data via buckets by user account on a shared device,
or by time frame if an application would like to prioritize last accessed
data.

- **Quota management**: applications can be smart with quota usage by keeping
track of quota usage per bucket and reserving quota before a write,
preventing errors when a user agent does not have enough disk space.
Application can also proactively choose to reduce low priority writes or
evict low priority buckets.

These use cases aim to help with the following class of applications that
store a lot of user data and aim to retrieve certain data without delay for a
smooth user experience.

- Email clients
- Video streaming services
- Music streaming services
- Document editors
- Applications that offer offline experience

## Setting up buckets

[The Storage Standard](https://storage.spec.whatwg.org/) already
[introduces buckets](https://storage.spec.whatwg.org/#buckets), but does not
have an API for explicitly managing buckets.

This explainer introduces the `navigator.storageBuckets.openOrCreate()` method.
Applications are expected to use this method to deliberately set up buckets
before using storage APIs. In the simplest form, usage looks as follows.

```javascript
// Create a bucket for emails that are synchronized with the server.
const inboxBucket = await navigator.storageBuckets.openOrCreate("inbox", {
  title: "Inbox",  // User agents may display titles in storage management UI.
});
```

Buckets can be assigned different storage policies at creation time. The example
below demonstrates some policies appropriate for data that is not (yet)
synchronized with a server. The policies introduced by this proposal will be
described in future sections.

```javascript
const draftsBucket = await navigator.storageBuckets.openOrCreate("drafts", {
    durability: "strict", persisted: true, title: "Drafts" });
```


## Getting the storage policies associated with a bucket

The storage policies passed to `openOrCreate()` are advisory. User agents may
create buckets whose policies don't match the requests. In most cases, the
deviations only result in different performance characteristics. Applications
can check a bucket's policies and take appropriate action when a vital policy
does not match the desired value.

```javascript
const draftsBucket = await navigator.storageBuckets.openOrCreate("drafts", {
    durability: "strict", persisted: true, title: "Drafts" });

if (await draftsBucket.persisted() !== true) {
  showWarningButterBar("Your drafts may be lost if you run out of disk space!");
}
```

Each `openOrCreate()` option that indicates a storage policy has a
corresponding property on the bucket object. Examples for all policy-related
properties will be shown in future sections.


## Accessing storage APIs from buckets

Each storage bucket has an entry point to the
[Cache storage API](https://w3c.github.io/ServiceWorker/#cache-objects). The
entry point matches `WindowOrWorkerGlobalScope.caches` in
[the Service Worker spec](https://w3c.github.io/ServiceWorker/#self-caches).

```javascript
const inboxCache = await inboxBucket.caches.open("attachments");
const draftsCache = await draftsBucket.caches.open("attachments");
```

Each storage bucket also has an entry point to the
[IndexedDB API](https://w3c.github.io/IndexedDB/). The entry point matches
`WindowOrWorkerGlobalScope.indexedDB` in
[the IndexedDB spec](https://w3c.github.io/IndexedDB/#factory-interface).

```javascript
const inboxDb = await new Promise(resolve => {
  const request = inboxBucket.indexedDB.open("messages");
  request.onupgradeneeded = () => { /* migration code */ };
  request.onsuccess = () => resolve(request.result);
  request.onerror = () => reject(request.error);
});
const draftsDb = await new Promise(resolve => {
  const request = draftsBucket.indexedDB.open("messages");
  request.onupgradeneeded = () => { /* migration code */ };
  request.onsuccess = () => resolve(request.result);
  request.onerror = () => reject(request.error);
});
```

Each storage bucket also has an entry point to
[the File API](https://w3c.github.io/FileAPI/). The entry points are
asynchronous versions of
[the Blob constructor](https://w3c.github.io/FileAPI/#dom-blob-blob) and
[the File constructor](https://w3c.github.io/FileAPI/#dom-file-file). File API
object created in a bucket are charged against the bucket's quota.

```javascript
const draftBlob = await draftsBucket.createBlob(
    ["Message text."], { type: "text/plain" });
const draftFile = await draftsBucket.createFile(
    ["Attachment data"], "attachment.txt",
    { type: "text/plain", lastModified: Date.now() });
```

TODO: Update the text here with the resolution of
https://github.com/w3c/FileAPI/issues/157.

Each storage bucket also has an entry point to the origin-private file system
in the [Native File System API](https://wicg.github.io/native-file-system/).
The entry point matches `WindowOrWorkerGlobalScope.getOriginPrivateDirectory()`
in [the Native File System spec](https://wicg.github.io/native-file-system/#sandboxed-filesystem).

```javascript
const inboxTestDir = await inboxBucket.getOriginPrivateDirectory();
const draftsTestDir = await draftsBucket.getOriginPrivateDirectory();
```

TODO: Update the text here with the resolution of
https://github.com/WICG/native-file-system/issues/210.


## Deleting buckets

Storage buckets can be deleted. For example, the code below could be used to
delete all the data stored on the device when the user logs out.


```javascript
await navigator.storageBuckets.delete("inbox");
await navigator.storageBuckets.delete("drafts");
```

A bucket's data becomes inacessible by the time the deletion operation
completes. For example, when deleting a bucket, all its IndexedDB databases will
be force-closed.


## Enumerating buckets

An origin can get a list of all its storage buckets.

```javascript
const bucketNames = await navigator.storageBuckets.keys();
console.log(bucketNames);  // [ "drafts", "inbox" ]
```

This function is provided for debugging / logging purposes, and may have
significant performance implications.


## Storage policy: Persistence

The storage specification currently endows each bucket with a
[mode](https://storage.spec.whatwg.org/#bucket-mode), which can be `persistent`
or `best-effort`. A persistent bucket will not be evicted without user notice
when the user agent experiences
[storage pressure](https://storage.spec.whatwg.org/#persistence). User agents
may also distinguish persistent buckets in their storage management UIs. For
example, Chrome presents an additional warning when a user chooses to delete
persistent storage.

This explainer proposes replacing the internal mode concept with a persistence
policy, in the interest of arriving at a uniform model for bucket behaviors. We
also propose the following API for operating on a bucket's persistence policy.

A bucket's persistence policy is specified at bucket creation time.

```javascript
const draftsBucket = await navigator.storageBuckets.openOrCreate("drafts", {
    persisted: true, title: "Drafts" });
```

The persistence policy can be queried at any time. The user agent may decline a
`persisted: true` policy requested by `openOrCreate()`.

```javascript
if (await draftsBucket.persisted() !== true) {
  showButterBar("Your email drafts may be lost if you run out of disk space");
}
```

The application can attempt to make a bucket persistent. The user agent may
decline the request.

```javascript
if (await draftsBucket.persisted() !== true) {
  const butterBar = showButterBar(
      "Your email drafts may be lost if you run out of disk space. Fix?");
  butterBar.onFixClicked = () => {
    if (await draftsBucket.persist() === true) {
      butterBar.hide();
      showButterBar("Your drafts are now safe! \o/");
    }
  }
}
```


## Storage policy: Durability

A bucket's durability policy is a hint that helps the user agent trade off
write performance against a reduced risk of data loss in the event of power
failures.

The policy has the following values.

* `"strict"` buckets attempt to minimize the risk of data loss on power failure.
  This may come at the cost of reduced performance, meaning that writes may take
  longer to complete, might impact overall system performance, may consume more
  battery power, and may wear out the storage device faster.

* `"relaxed"` buckets may "forget" writes that were completed in the last few
  seconds, when a power loss occurs. In return, writing data to these buckets
  may have better performance characteristics, and may allow a battery charge
  to last longer, and may result in longer storage device lifetime. Also,
  power failures will *not* lead to data corruption at a higher rate than for
  `"strict"` buckets.

In general, `"strict"` buckets are intended to store data created by the user
that has not been synchronized with the application's server. In this case, the
application would not be able to recover from power loss. By contrast,
`"relaxed"` buckets are most suitable for caches that can be repopulated easily.

A bucket's durability policy is specified at bucket creation time.

```javascript
const draftsBucket = await navigator.storageBuckets.openOrCreate("drafts", {
    durability: "strict", title: "Drafts" });
```

The durability policy can be queried at any time. The user agent may not
honor the policy requested by the `openOrCreate()` call.

```javascript
if (await draftsBucket.durability() !== "strict") {
  showButterBar("Your email drafts may be lost if you run out of power");
}
```

A bucket's durability policy cannot be changed once the bucket is created.


## Getting a bucket's quota usage

In order to support eviction at bucket granularity, user agents are expected to
track quota usage for the data associated with each bucket. Applications can get
this information using an API similar to
[StorageManager.estimate()](https://storage.spec.whatwg.org/#dom-storagemanager-estimate).

```javascript
const inboxEstimate = await inboxBucket.estimate();
if (inboxEstimate.usage >= inboxEstimate.quota * 0.95) {
  displayWarningButterBar("Go to settings and sync fewer days of email");
}
```

## Reserving quota for a bucket

TODO: Flesh out this section or move it into a separate explainer.

```javascript
const autosaveBucket = await navigator.storageBuckets.openOrCreate(
  "autosave",
  {
    title: "Autosaved Form Data",
    durability: "strict",
  });
await autosaveBucket.reserve(20 * 1024 * 1024);  // 20 MB
```


## The default bucket

The following existing APIs operate on the `default` bucket. These APIs create
the `default` bucket on-demand.

* `WindowOrWorkerGlobalScope.indexedDB` in
  [IndexedDB](https://w3c.github.io/IndexedDB/#factory-interface)
* `WindowOrWorkerGlobalScope.caches` in
  [ServiceWorker](https://w3c.github.io/ServiceWorker/#self-caches)
* `NavigatorStorage.storage` in
  [the Storage Standard](https://storage.spec.whatwg.org/#api)
* `WindowOrWorkerGlobalScope.getOriginPrivateDirectory` in
  [Native File System](https://wicg.github.io/native-file-system/#sandboxed-filesystem)

The default bucket is created with the following options.

```js
await navigator.storageBuckets.openOrCreate("default", {
  durability: "strict", persist: false, title: "" });
```


## Storage buckets and service workers

TODO: Flesh out this section or move it into a separate explainer.

Each storage bucket can store
[service worker registrations](https://w3c.github.io/ServiceWorker/#dfn-service-worker-registration).
When a storage bucket is deleted, all service worker registrations that it
contains are also evicted.

Applications are expected to associate a service worker registration with a
storage bucket if the service worker would not be able to do its job without
the data in the bucket. This way the user agent won't have to spend system
resources on waking up a service worker, only to find out that the service
worker cannot fulfill the request given to it.

```javascript
const inboxRegistration = await inboxBucket.serviceWorker.register(
    "/inbox-sw.js", { scope: "/inbox" });
const draftsRegistration = await draftsBucket.serviceWorker.register(
    "/drafts-sw.js", { scope: "/drafts" });
```

Storage buckets expose access to their service workers via the following subset
of the
[ServiceWorkerContainer](https://w3c.github.io/ServiceWorker/#serviceworkercontainer)
methods.

* [register](https://w3c.github.io/ServiceWorker/#dom-serviceworkercontainer-register)
* [getRegistration](https://w3c.github.io/ServiceWorker/#dom-serviceworkercontainer-getregistration)
* [getRegistrations](https://w3c.github.io/ServiceWorker/#dom-serviceworkercontainer-getregistrations)


## Storage buckets and the Clear-Site-Data

TODO: Flesh out this section or move it into a separate explainer.

[Clear-Site-Data](https://w3c.github.io/webappsec-clear-site-data/) currently
supports deleting all DOM-accessible storage via the `storage` type. We propose
adding the ability to clear individual buckets using a family of types looking
like `storage:bucket-name`.

For example, receiving an HTTP response with the following header would cause
the `inbox` bucket to be deleted.

```
Clear-Site-Data: "storage:inbox"
```


## Key Scenarios

This is a list of scenarios which we hope buckets will help applications solve.

### Storage Eviction

Currently during storage eviction, the browser will delete the entire
origin’s data. This can cause a broken experience for users and does not
allow for applications to best mitigate this poor experience. Buckets aims to
give applications more control of prioritizing storage during eviction and
decide on these tradeoffs themselves under storage pressure.

In this example, a document client will use Buckets to set different
priorities for different types of documents during storage pressure. The
`recent` bucket stores recently accessed documents that have already been
uploaded to the server but have been cached locally for easy user access. The
`drafts` bucket stores drafts of documents that have been made offline, which
have not yet but uploaded to the server and are irrecoverable if lost.

In this scenario, the `drafts` bucket is created with `persisted: true` and
`durability: "strict"` to specify that it should be evicted last upon storage
pressure, and all data should survive power failures at the cost of more
battery consumption. The `recent` bucket will be evicted first because of its
low priority since it contains data that can be recovered from the server.

TODO: Add image

```javascript
const recentBucket = await navigator.storageBuckets.openOrCreate("recent",
    { durability: "relaxed", persisted: false, title: "Recent" });

const draftsBucket = await navigator.storageBuckets.openOrCreate("drafts",
    { durability: "strict", persisted: true, title: "Drafts" });
```

### Partitioned Storage

Currently there isn’t an option for granular partitioning of stored data
making it hard for applications to evict all relevant low priority data at
once. Buckets allow applications to partition data however they seem fit.

In this example we will explore a scenario where an email client will
partition storage using Buckets by user accounts on a shared device.

TODO: Add image

```javascript
// Bucket creation per user account.
const user1Bucket = await navigator.storageBuckets.openOrCreate(
    "userid111111_inbox", {title: "alice@email.com Inbox" });

const user2Bucket = await navigator.storageBuckets.openOrCreate(
    "userid222222_inbox", {title: "bob@email.com Inbox" });

```

Data can be stored for each user via Storage APIs per bucket without
commingled user data.

```javascript
// Attachments for User 1
const attachments = user1Bucket.caches.open("attachments");
// Messages for User 2
const req = user2Bucket.indexedDB.open("inbox", 1);
… // Retrieving cached inbox messages.
```

When Bob does not login for 30 days on the device, an application can easily
delete all storage associated with Bob, while keeping data from Alice
undisrupted.

```javascript
await navigator.storageBuckets.delete("user2_inbox");
```


### Quota Management

The following example shows how a video streaming service can use Buckets and
its quota APIs to assign storage limits and reserve quota for individual
buckets to better control their quota usage and reserve space for certain
data.

In this example, Bucket A will store videos users have explicitly downloaded
onto their device for offline access. Bucket B will store data for user
recommendations. The service can decide that they want to reserve the quota
usage for Bucket B, saving more space for Bucket A.

TODO: Add image and details on reserve quota.

```javascript
const offlineVideosBucket = await navigator.storageBuckets.openOrCreate(
    "offline_videos",
    { title: "Offline Videos", durability: "strict", persisted: false });

const recommendationBucket = await navigator.storageBuckets.openOrCreate(
   "recommendations", {title: "Recommendations" });
```

Reserve quota to ensure storage availability for recommended videos bucket.

```javascript
await recommendationBucket.reserve(20 * 1024 * 1024);  // 20 MB
```

Query quota usage to check if a user can download more videos.

```javascript
const offlineVideosEstimate = await offlineVideosBucket.estimate();
if (offlineVideosEstimate.usage >= offlineVideosEstimate.quota * 0.95) {
  displayWarningButterBar("Delete old downloaded videos to download more");
}
```

## Detailed design discussion

### Bucket names

We propose restricting developer-facing bucket names to strings made up of a
very small set of characters.

Here are the characters we propose allowing.

* lowercase Latin letters `a` - `z`
* digits `0` - `9`
* the special characters `-` and `_` can be used in the middle of the name, but
  not at the beginning

The restrictions were chosen with two goals in mind.

1. Avoid gotchas when including a bucket name in a Clear-Site-Data header.
   Embedded unrestricted strings in HTTP headers requires escaping. Developers
   unaware of the limitations (or pressed by deadlines) may use simple string
   interpolation instead of proper escaping. Errors may not be caught early
   on, if initial bucket names happen to be safe to include in headers.

2. Give browsers the option to integrate bucket names in file names on the
   computer's file system. This may help user agents avoid a database lookup in
   their `createOrOpen()` implementations. We expect that opening buckets will
   end up on the critical path for loading modern sites, so we consider that
   giving user agents maximum freedom in the name of efficiency serves users,
   which outweighs developer convenience.

The desire for direct integration into file system names significantly
constrains the character set. The constraints we are aware of are listed below.

* Characters outside the ASCII set may cause encoding-related problems.
* Non-printable characters may be disallowed by some file systems.
* Allowing both uppercase and lowercase characters would cause problems on
  case-insensitive file systems.
* Many non-alphanumeric characters have special meaning on some file systems.
  For example, `.` separates file names from extensions, and files ending in
  `.exe` are executable on Windows.

The `_` character in bucket names is designated for conveying hierarchical
structure. The user agent may choose to display this structure in its storage
management UI.

For example, assume three buckets with names `user123456`, `user123456_inbox`
and `user123456_drafts`, and titles `pwnall@chromium.org`, `Inbox` and `Drafts`.
These buckets may be displayed in the UI as follows.

* pwnall@chromium.org
  * Inbox
  * Drafts

Bucket names are limited to 64 characters. This supports implementations that
would directly integrate bucket names into file names, and makes it easy to
reason about bucket lookup performance.


### Bucket titles

Buckets are expected to be named using programmer-friendly identifiers, simiarly
to variable names. By contrast, bucket titles are user-friendly descriptions.
Titles are intended to support user agents that want to offer the ability to
delete individual buckets in their storage management UI. These user agents may
display bucket titles when showing buckets to their users.

`title` was chosen for consistency with the
[HTML title element](https://html.spec.whatwg.org/multipage/semantics.html#the-title-element).

Bucket titles present some subtleties for applications that support multiple
languages. Specifically, a bucket's title will presumably reflect the
users' preferred language at the time the bucket is created. The current API
surface does not allow changing a bucket's description, so already-created
buckets will not reflect language preference changes.

Including application strings in user agent UI has non-trivial security and
privacy implications, which may deter some user agents from using the `title` as
intended. For example, user agents that intend to incorporate the `title` need
to mitigate against misleading values such as
`"You have a virus! Go to www.evil.com for a cleanup"`.


### Storage policy naming

Here are the considerations used for storage policy naming.

`persisted` was chosen for consistency with
[StorageManager.persisted()](https://storage.spec.whatwg.org/#dom-storagemanager-persisted)
in the Storage specification. The true / false values are also consistent with
the definitions in the Storage specification.

`durability` was chosen for consistency with
[IDBTransaction.durability](https://w3c.github.io/IndexedDB/#dom-idbtransaction-durability)
in IndexedDB. The values `"strict"` and `"relaxed"` have the same significance
as the corresponding
[IndexedDB transaction hints](https://w3c.github.io/IndexedDB/#transaction-durability-hint).


### Durability guarantees

Data "written" by a storage API conceptually goes through the following four
stages.

1) The data starts out cached in an application-level buffer. If the current tab
   or the entire user agent crashes (due to a bug or resource exhaustion), the
   data is lost.

2) The data is flushed to an OS-level (usually kernel) buffer. At this point,
   the data will survive a tab or user agent crash. However, the data is lost
   if the entire operating system crashes ([blue screen of
   death](https://en.wikipedia.org/wiki/Blue_screen_of_death) on Windows,
   [kernel panic](https://en.wikipedia.org/wiki/Kernel_panic) on UNIX
   systems).

3) The data is flushed to a storage device (HDD / SSD) buffer. At this point,
   the data will survive an OS crash. However, if the computer experiences a
   power failure, the data may be lost. Some storage devices have batteries with
   sufficient capacity to write the data in the buffers in case of power
   failure, but this is not generally guaranteed. Furthermore, most modern
   portable computers (laptops, tablets, mobile phones) have firmware / OS logic
   that mitigates power failures by suspending the computer's activity and
   writing all the data in volatile buffers.

4) The data is persisted to the storage medium (disk platters for HDDs,
   non-volatile memory cells for SSDs). At this point, the data will surive a
   computer power failure.

Storage systems may differ in how they handle a write operation, such as
commiting an
[IndexedDB transaction](https://w3c.github.io/IndexedDB/#transaction-construct),
along the following axes.

1) The stage at which the system reports that the write has completed. For
   example, many database systems offer the option of considering that a
   transaction has committed when the data is flushed to OS-level buffers.

2) The stage to which the data is pushed after the write has completed. In the
   example above, after a database system considers a transaction to have
   completed, it may ask the OS to flush the data to the storage
   device, or it can let the data stay in OS-level buffers until the OS decides
   to evict the data to the storage device.

Combining the stage at which a write is reported as completed with the stage to
which the data is pushed results in 12 possibile behaviors for storing data. The
behaviors with a higher risk of data loss result in better performance along a
few axes such as write speed, battery usage, and storage medium wear.

The storage buckets API narrows down this complex space to only two options,
which are packaged as possible values for the `durability` policy.

* The `strict` durability policy requires that the data is persisted to the
  storage medium before writes are considered to have completed. This policy
  results in lower performance, but guarantees that data will survive power
  losses. Therefore, this policy is  the right choice for user data that cannot
  be recovered from an alternative source in the event of a power failure.

* The `relaxed` durability policy requires that the data is flushed to OS-level
  buffers below writes are considered to have completed, and allows the data to
  remain in OS-level buffers for an indefinite amount of time. This policy
  results in better performance, at the cost of risking data loss. For this
  reason, this policy is intended for data that can be easily obtained from an
  alternative source, such as cached versions of data stored on the
  application's server.

We drew inspiration from [SQLite](https://www.sqlite.org/) and
[LevelDB](https://github.com/google/leveldb), which are the two libraries used
by the browsers that are popular at the time of this writing. The following
facts were considered by our decision process.

* SQLite allows choosing between flushing data to the operating system
  (similarly to the `relaxed` policy), and flushing it to the storage device or
  media (similarly to the `strict` policy) via the
  [synchronous PRAGMA](https://www.sqlite.org/pragma.html#pragma_synchronous).
  This setting's behavior is closely connected to whether a database uses
  [Write-Ahead Logging (WAL)](https://www.sqlite.org/wal.html) or not, which
  must be decided when the database is open via the
  [journal_mode PRAGMA](https://www.sqlite.org/pragma.html#pragma_journal_mode).

* SQLite allows choosing between flushing to the storage device and flushing to
  the media via the
  [fullfsync PRAGMA](https://www.sqlite.org/pragma.html#pragma_fullfsync) and
  the
  [checkpoint_fullfsync PRAGMA](https://www.sqlite.org/pragma.html#pragma_checkpoint_fullfsync).

* LevelDB allows choosing between the `relaxed` policy and the `strict` policy
  at transaction level via the
  [WriteOptions.sync option](https://github.com/google/leveldb/blob/master/doc/index.md#synchronous-writes).


## Considered alternatives

[This should include as many alternatives as you can, from high level
architectural decisions down to alternative naming choices.]

### Expose the API off of navigator.storage.buckets

Instead of exposing the storage buckets API off of `navigator.storageBuckets`,
we have exposed it off of `navigator.storage.buckets`. The examples below
illustrate this alternative.

```javascript
const inboxBucket = await navigator.storage.buckets.openOrCreate("inbox", {
  title: "Inbox",
});
const draftsBucket = await navigator.storage.buckets.openOrCreate("drafts", {
  durability: "strict", persisted: true, title: "Drafts" });

await navigator.storage.buckets.delete("inbox");
await navigator.storage.buckets.delete("drafts");
```

`navigator.storage.buckets` was rejected to avoid developer confusion around
nesting and the default bucket. Specifically, some `navigator.storage` methods
(`estimate()`, `persist()` and `persisted()`) operate on the default bucket, so
`navigator.storage` seems connected to the default bucket.
`navigator.storage.buckets` is a property on `navigator.storage`, and it may be
confusing that it refers to all the origin's buckets, not to the default bucket.


### Separate intents for creating a bucket and opening an existing bucket

`navigator.storageBuckets.openOrCreate()` always attempts to create a bucket
with the given name if it does not exist. This is different from storage APIs on
most systems, where developers can express the three separate intents below.

1) Open the named bucket if it exists, create it if it does not exist.
2) Open the named bucket if it exists, fail if it does not exist.
3) Fail if the named bucket exists, create it if it does not exist.

Allowing all three intents to be expressed could have been accomplished by
having separate methods (`openOrCreate()`, `open()`, `create()`), as
illustrated in the example below.

```javascript
// Creates the "inbox" bucket if does not already exist.
const inboxBucket = await navigator.storageBuckets.openOrCreate("inbox", {
  title: "Inbox" });

// Fails if the "inbox" bucket does not already exist.
const inboxBucket = await navigator.storageBuckets.open("inbox", {
  title: "Inbox" });

// Fails if the "inbox" bucket already exists.
const inboxBucket = await navigator.storageBuckets.create("inbox", {
  title: "Inbox" });
```

Alternatively, we could have settled for one `open()` method with options, as
shown below.

```javascript
// By default, creates the "inbox" bucket if does not already exist.
const inboxBucket = await navigator.storageBuckets.open("inbox", {
  title: "Inbox" });

// Fails if the "inbox" bucket does not already exist.
const inboxBucket = await navigator.storageBuckets.open("inbox", {
  title: "Inbox", failIfNotExist: true });

// Fails if the "inbox" bucket already exists.
const inboxBucket = await navigator.storageBuckets.open("inbox", {
  title: "Inbox", failIfExists: true });
```

We think that only exposing the `openOrCreate()` option is the best way to
support the model where each bucket can be evicted by the browser independently
of other buckets. We want applications to be written assuming that each time
they attempt to open a bucket, they may be creating the bucket from scratch.


### Bucket designations for each storage API

Instead of exposing entry points for each API on the bucket object, we could add
integration points for buckets to each storage API. Examples below.

```javascript
const inboxCache = caches.open("attachments", { bucket: "inbox" });

const inboxDb = await new Promise(resolve => {
  const request = inboxBucket.indexedDB.open("messages", { bucket: "inbox" });
  request.onupgradeneeded = () => { /* migration code */ };
  request.onsuccess = () => resolve(request.result);
  request.onerror = () => reject(request.error);
});

const inboxBlob = new Blob(
    ["Message text."], { type: "text/plain", bucket: "inbox" });
const inboxFile = new File(
    ["Attachment data"], "attachment.txt",
    { type: "text/plain", lastModified: Date.now(), bucket: "inbox" });

const inboxTestDir = await self.getOriginPrivateDirectory({ bucket: "inbox" });

const inboxRegistration = await navigator.serviceWorker.register(
    "/inbox-sw.js", { scope: "/inbox", bucket: "inbox" });
```

TODO: Explain why this is worse than the main decision.


### Allow all safe characters for HTTP headers in bucket names

TODO: Explain that the current setup puts users above developers, but the
performance benefit is pretty weak. Summarize safe transformation scheme that
does not require a database lookup.


### No length restriction for bucket names

Bucket names are currently limited to 64 characters. We could remove this
limitation, and rely on the ecosystem to evolve its own rules. User agents would
most likely provide some guidance that would evolve based on developer needs.

The alternative of not having any length restriction at all was rejected
because our previous experience strongly suggests the need for guardrails.

IndexedDB does not (at the time of this writing) have limitations on key
sizes, and some developers did try using very large (multi-megabyte) keys.
IndexedDB implementations use these keys as primary keys in a database, and
large primary keys have lead to out-of-memory crashes and performance cliffs.

In summary, in the absence of immediate feedback at the prototyping stage,
developers will use very large identifiers, even if that may result in a bad
user experience.


### Relaxed length restrictions for bucket names

Bucket names are currently limited to 64 characters. This restriction may
inconvenience developers, especially if deep hierarchies are in use. For
example, `user123456789_inbox_label123456789` has 34 characters, which uses more
than 50% of the length budget. Relaxing the name lenth limit to 1024 characters
would remove developer concerns.

This alternative was rejected in the interest of avoiding unnecessary
complexity. We are aware of implementation techniques for handling
1024-character bucket names efficiently, at the cost of some extra complexity in
the user agent. This extra complexity still costs the user battery power (more
code is being fetched and executed), as well as data transfer and storage
(increased binary size). It will be far easier to increase the length limit
(64 -> 1024) down the line than it would be reduce it (1024 -> 64), so we are
proposing the stricter limit until we see concrete developer needs.

When reasoning about the potential increase in complexity, we considered the
following implementation alternatives.

1. Rely on the underlying storage to handle the 1024-character keys. All major
   underlying systems we know (file system, SQLite, LevelDB) work better with
   small names. SQLite, LevelDB, and some file systems can handle 1024-character
   names.

2. Use a fast string hash, such as [xxHash](https://github.com/Cyan4973/xxHash)
   to map potentially long bucket names to very short hashed names. The hashed
   names are guaranteed to be very short, for example xxHash produces 8/16-byte
   hashes. However, fast string hashes may produce collisions, and allowing
   collisions will result in more complex storage code.

3. Use a strong cryptographic hash, such as
   [SHA-256](https://en.wikipedia.org/wiki/SHA-2). Compared to fast string
   hashes, strong cryptographic hashes are larger (SHA-256 produces 32-byte
   outputs) and significantly slower. In return, cryptographic hashes
   guarantee vanishingly small collision rates, so the implementation does
   not need to handle collisions. We expect this to be the preferred
   alternative for handling long bucket names, because the overheads are
   small compared to the reduced complexity. Modern computers (both desktop
   and mobile) have hardware accelerators for computing cyptographic hashes,
   and hash output sizes are within the range of key sizes that yield good
   database performance.

Relaxing the length to 1024 characters makes it less likely that bucket names
can be


### Alternative name for the bucket `title` property

The `title` properties could be named `description` instead. This would be
consistent with the
[Web App Manifest description](https://w3c.github.io/manifest/#description-member).

We preferred `title` to `description` because we think that `title` invites
developers to be brief (1-3 words) whereas the latter appears to ask for a
full sentence. We expect that short strings will be better suited for inclusion
in user management UIs.


### Language maps for bucket titles

The bucket `title` property could allow a dictionary instead of a string, where
the keys are valid values for the
[lang attribute in the HTML specification](https://html.spec.whatwg.org/#the-lang-and-xml:lang-attributes),
and values are localized user-facing strings.

TODO: The translations and codes need to be checked.

```javascript
const draftsBucket = await navigator.storageBuckets.openOrCreate("drafts", {
  durability: "strict", persisted: true,
  title: { en: "Drafts", es: "Borradoers", jp: "下書き" }});
```

### Enumerate all buckets using an async iterator

The function used to enumerate buckets returns a sorted array of bucket names.
It could have been specified to return an asynchronous iterator.

```javascript
const bucketNames = [];
for await (let name of navigator.storage.buckets.keys())
  bucketNames.push(name);

console.log(bucketNames);  // [ "drafts", "inbox" ]
```

The main benefit of using an asynchronous iterator would be the ability to scale
to a very large number of buckets.

This alternative was rejected because we expect that the number of buckets
created by origins will not be large enough to run into memory limitations. In
order to support per-bucket eviction, user agents will likely need


### Separate durability options for flushing to the storage device vs media

The `durability` bucket property currently supports the storage policies
`"relaxed"` and `"strict"`. Instead of the `"strict"` policy, we could have
provided separate policies for flushing the data to storage device buffers
(`"device"`) and for flushing the data to storage media (`"media"`).

Under this alternative, advanced applications would take advantage of the
extra flexibility for increased performance, while simpler applications would
use `"media"`, which is equivalent to `"strict"`. The example below outlines
the code needed to use the extra storage policy, and is suggestive of the
additional complexity introduced by this alternative.

```javascript
// Writes to this bucket are flushed to the storage device. The data here may be
// lost in case of power failure.
//
// Each change (keystroke) in a draft is saved here.
const immediateDraftsBucket = await navigator.storage.buckets.openOrCreate(
    "device-drafts",
    { durability: "device", persisted: true, title: "Drafts Cache" });
const immediateDraftsDb = await new Promise(resolve => {
  const request = inboxBucket.indexedDB.open("messages", { bucket: "inbox" });
  request.onupgradeneeded = () => { /* migration code */ };
  request.onsuccess = () => resolve(request.result);
  request.onerror = () => reject(request.error);
});

// Writes to this bucket are flushed to the storage media. The data here will
// not be lost on power failure.
//
// Draft changes are batched every minute and saved here. Writing to this bucket
// on every keystroke is too much of a battery drain.
const draftsBucket = await navigator.storage.buckets.openOrCreate(
    "media-drafts", { durability: "media", persisted: true, title: "Drafts" });
const draftsDb = await new Promise(resolve => {
  const request = inboxBucket.indexedDB.open("messages", { bucket: "inbox" });
  request.onupgradeneeded = () => { /* migration code */ };
  request.onsuccess = () => resolve(request.result);
  request.onerror = () => reject(request.error);
});

// Accumulates changes that have been stored in immediateDraftsDb but not in
// draftsDb.
const batchedDrafts = [];

// Called on every draft change, which may happen on every user key stroke or
// mouse click.
async function saveDraft(draft) {
  batchedDrafts.push(draft);

  const transaction = immediateDraftsDb.transaction("messages", "readwrite");
  const messageStore = transaction.objectStore("messages");
  await new Promise((resolve, reject) => {
    objectStore.put(draft);
    transaction.commit();
    transaction.oncomplete = resolve;
    transaction.onerror = () => reject(transaction.error);
  });
}

// Called every minute to write new draft changes to the persistent database.
async function flushDrafts() {
  if (batchedDrafts.length === 0)
    return;

  // Swap the drafts queue to avoid double flushing.
  const drafts = batchedDrafts;
  batchedDrafts = [];

  // TODO: Eliminate redundant draft changes.

  const transaction = db.transaction("messages", "readwrite");
  const messageStore = transaction.objectStore("messages");
  await new Promise((resolve, reject) => {
    for (let draft of drafts)
      objectStore.put(draft);
    transaction.commit();
    transaction.oncomplete = resolve;
    transaction.onerror = () => reject(transaction.error);
  });
}

setInterval(flushDrafts, 60 * 1000);
```

The alternative was rejected because we considered that the extra complexity is
not worth the performance benefits. Specifically, buckets using the `"media"`
policy would allow the storage device to batch the writes its internal buffers.
Under our proposed design, buckets that would have used the `"media"` policy
will be indistinguishable from buckets that would have used the `"device'`
policy, the writes to these buckets will be flushed directly to storage media.
We are foregoing the following advantages.

* Writes that must go straight to the storage media reduce the flexibility of
  the on-device I/O scheduler. This may impact the performance of seemingly
  unrelated I/O requests.

* Writing to the storage media more often will wear out the media, reducing the
  life span of the storage device. This effect may be more significant in the
  lower end of the market, where devices have lower endurance specifications.

* Writing to the storage media more often will consume extra battery power on
  mobile computers.

We were also influenced by the fact that most operating systems that are
currently popular do not support the distinction between the two storage
policies. The points below outline the current state of OS support.

* On Linux-based systems, which include Android and ChromeOS,
  [fdatasync()](https://linux.die.net/man/2/fdatasync) flushes all the changes
  in a file from OS-level buffers to to the storage device media, which is
  consistent with the `"media"` storage policy.
  [fsync()](https://man7.org/linux/man-pages/man2/fsync.2.html) also flushes
  file metadata that is not strictly needed to read the data, such as file
  modification and access times.

* On Windows, the `CreateFile()` flag
  [FILE_FLAG_WRITE_THROUGH](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilea#:~:text=FILE_FLAG_WRITE_THROUGH)
  is fairly similar to the `"strict"` durability policy, as it requires that all
  writes are flushed to the storage device.
  [FlushFileBuffers()](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-flushfilebuffers)
  flushes a file's changes to the storage device. The function
  [flushes to media in Windows 8+](https://devblogs.microsoft.com/oldnewthing/20170510-00/?p=95505),
  which is consistent with the `"media"` policy. However, on Windows 7 and
  below, variations in device drivers may cause the function to only flush to
  storage device buffers, which is consistent with the `"device"` policy.

* On Darwin-based systems, which include macOS and iOS,
  [fsync()](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fsync.2.html)
  flushes a file's changes to the storage device buffers, which is consistent
  with the `"device"` storage policy. This behavior is compliant with the
  [POSIX 2018 specification for fsync()](https://pubs.opengroup.org/onlinepubs/9699919799/functions/fsync.html),
  which only demands that the data be "transferred to the storage device".
  Flushing changes to the storage media platters, required by the `"media"`
  policy, is accomplished by passing the
  [F_FULLFSYNC](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fcntl.2.html#:~:text=F_FULLFSYNC)
  flag to `fcntl()`.


### Separate durability option for application-level buffers

The `durability` bucket property currently supports the storage policies
`"relaxed"` and `"strict"`. Instead of the `"relaxed"` policy, we could have
provided separate policies for completing the write while the data is in
application-level buffers (`"app"`) and for flushing the data to OS-level
buffers (`"kernel"`).

At a high level, this alternative is about a tradeoff between offering
applications more flexibility, in the name of performance, and offering a
simpler storage model. So, the analysis here should be similar to the discussion
around durability options for flushing to the storage device vs media, covered
in the section above. However, this section is significantly shorter, because
the performance benefits are much less significant.

This section does not have sample code, because we could not find any use case
that would benefit from the separation proposed here. Instead, we'll discuss the
performance benefits offered by such a separation, and their relevance.

Writes to buckets with the `"app"` storage policy would be considered complete
as soon as the data has been serialized in an application-level buffer inside
the user agent. Not having to wait for the data to be flushed to OS-level
buffers has the following advantages.

* Flushing the data to the OS involves system calls, which perform CPU context
  switches. The context switches have non-trivial costs.

* Flushing the data to the OS may require copying the data buffers, which can be
  expensive when writing large amounts of data.

This alternative was rejected because these advantages above are considered to
be mostly theoretic, for the following reasons.

* All usage of storage APIs may require system calls. Modern user agents have
  multi-process architectures. Most storage API features, such as transactions,
  require [IPC](https://en.wikipedia.org/wiki/Inter-process_communication)
  between a process running the site's JavaScript and a coordinating process.
  IPC requires system calls.

* Data buffer copies may be avoided. All modern operating systems have shared
  memory features for avoiding large data copies during IPC. Modern operating
  systems also have direct I/O features that either allow the user agent to
  serialize data directly into an OS-level buffer, or allow applications to
  surrender ownership of buffers to the OS.

This decision was influenced the fact that application-level buffers are not
used in the built-in [SQLite VFS](https://www.sqlite.org/vfs.html)
implementations, in the built-in
[LevelDB Env](https://github.com/google/leveldb/blob/master/include/leveldb/env.h)
implementations, or in
[Chrome's file abstraction](https://source.chromium.org/chromium/chromium/src/+/master:base/files/file.h).
The main reason this option was consisdered is that
[fopen()](https://pubs.opengroup.org/onlinepubs/9699919799/functions/fopen.html)
in the POSIX standard is specified to use an application-level buffer, which
may be flushed to an OS-level buffer using
[fflush()](https://pubs.opengroup.org/onlinepubs/9699919799/functions/fflush.html).


### Default to strict durability

Newly created buckets receive the `"relaxed"` storage policy, unless a different
`durability` option is passed to `openOrCreate()`. The default storage policy
could have been `"strict"`. This alternative was rejected for two reasons,
outlined below.

First, we expect that Web applications will mostly use client-side storage to
cache data, where the authoritative copy is stored on the application server.
This use case is best served by the `"relaxed"` policy. The example below shows
that this alternative leads to more bulky code for expressing the common case.

```javascript
const inboxBucket = await navigator.storageBuckets.openOrCreate("inbox", {
  title: "Inbox", durability: "relaxed" });

const draftsBucket = await navigator.storageBuckets.openOrCreate("drafts", {
  persisted: true, title: "Drafts" });
```

Second, we think that the current proposal will make it easier to reason about
correctness in a code review. Reviewers can assume that code that explicitly
mentions `durability: "strict"` is operating on data that must survive power
outages, and can focus on recovery logic for this code. If we followed the
alternative, reviewers that encounter a bucket without a `durability` option
would have to check if the author forgot to specify `durability: "relaxed"`,
if the bucket stores  data that cannot be recovered from another source.

TODO: Explain that durability guarantees apply to individual writes, but
applications need to handle inconsistencies across storage APIs (Cache
Storage and IndexedDB). We don't offer two-phase commit across storage APIs.


### Support changing a bucket's durability policy

Once a bucket is created, the value of its `durability` policy is fixed. This is
inconsistent with the `persisted` policy, which can be changed after the bucket
is created via the `persist()` method.

If storage buckets allowed changing the `durability` policy, applications could
switch policies based on dynamically changing conditions. The email client in
our example might want to allow the user to switch between storing email drafts
with `"strict"` durability and storing the drafts with `"relaxed"` durability.

```javascript
const draftsBucket = await navigator.storageBuckets.openOrCreate("drafts", {
  durability: "strict", persisted: true, title: "Drafts" });

// Called when the user switches a "drafts" durability toggle.
async function setDraftsDurability(durability) {
  // If durability can change, it definitely needs to be exposed using an async
  // function.
  const currentDurability = await draftsBucket.durability();
  if (currentDurability === durability)
    return;

  await draftsBucket.setDurability(durability);
}
```

In the world proposed by this explainer, the email client could offer the same
flexibility to the user at the cost of extra complexity. The email client would
create two buckets, write drafts to the bucket that reflects the user's current
preferences, and read drafts from both buckets.

```javascript
const draftsBuckets = {};
draftsBuckets.strict = await navigator.storageBuckets.openOrCreate("drafts", {
  durability: "strict", persisted: true, title: "Drafts (Durable)" });
draftsBuckets.relxaed = await navigator.storageBuckets.openOrCreate("drafts", {
  durability: "relaxed", persisted: true, title: "Drafts (Fast)" });


const draftsDb = {};
for (let durability of ["relaxed", "strict"]) {
  draftsDb[druability] = await new Promise(resolve => {
    const request = draftsBucket[durability].indexedDB.open("messages");
    request.onupgradeneeded = () => { /* migration code */ };
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}

// Called on every draft change, which may happen on every user key stroke or
// mouse click.
async function saveDraft(draft) {
  batchedDrafts.push(draft);

  const transaction = immediateDraftsDb.transaction("messages", "readwrite");
  const messageStore = transaction.objectStore("messages");
  await new Promise((resolve, reject) => {
    objectStore.put(draft);
    transaction.commit();
    transaction.oncomplete = resolve;
    transaction.onerror = () => reject(transaction.error);
  });
}

// TODO: Code that reads drafts should operate on both databases.
```

This alternative was rejected because of concerns that it would significantly
reduce the degrees of freedom of user agent implementations, which could result
in reduced performance for all applications.

For example, let's consider a user agent that only targets Linux-based
systems and relies on SQLite to implement storage APIs. This user-agent could
implement `"strict"` buckets using SQLite databases (or one consolidated
database per origin) with
[PRAGMA synchronous](https://www.sqlite.org/pragma.html#pragma_synchronous) set
to `FULL` and
[PRAGMA journal_mode](https://www.sqlite.org/pragma.html#pragma_journal_mode)
set to `DELETE`, in order to take advantage of
[SQLite's F2FS fast path](https://www.sqlite.org/compile.html#enable_batch_atomic_write).
`"relaxed"` buckets use SQLite databases with PRAGMA synchronous set to `DELETE`
and PRAGMA journal_mode set to `WAL`, which would take advantage of
[WAL mode](https://www.sqlite.org/wal.html).

The example above illustrates that user agents may be able to obtain better
performance if they can place data with different `durability` policies in
entirely different underlying stores. The ability to change a bucket's
`durability` policy would significantly undermine this flexibility.


### Synchronous access to a bucket's storage policies

Bucket storage policies are currently accessible using methods that return
Promises. This information could have been exposed using (synchronous)
properties instead.

```javascript
if (draftsBucket.persisted !== true) {
  showButterBar("Your email drafts may be lost if you run out of disk space");
}
if (draftsBucket.durability !== "strict") {
  showButterBar("Your email drafts may be lost if you run out of power");
}
```

Exposing synchronous access to the persistence policy was rejected due to
complications stemming from the ability to change a bucket's persistence policy
after the bucket is created. Our options for handling changes would be to
either say that a bucket's `persisted` property reflects the policy at the time
the bucket is opened, or say that the `persisted` property magically changes
when the policy changes. Both options seem confusing for developers.

Exposing synchronous access to the durability policy was rejected in the
interest of maxmizing the consistency of the API across storage policies.


### Keep deleted buckets alive while there are references to them

TODO: Explain that this is bad for predictability. We want the data to be gone
when we say "logout completed", and we want to return quota when a bucket is
deleted.


### Integrate storage buckets with DOM Storage

The integration points listed in this explainer intentionally exclude the
[Web Storage API](https://html.spec.whatwg.org/multipage/webstorage.html). We
could have included
[localStorage](https://html.spec.whatwg.org/multipage/webstorage.html#the-localstorage-attribute)
on the list of APIs that buckets offer.

```javascript
const settingsBucket = await navigator.storageBuckets.openOrCreate("settings", {
  title: "User Preferences",
});

const emailsPerPage = settingsBucket.localStorage.getItem('emailsPerPage');
```

This alternative was rejected because we were concerned about the performance
implications of supporintg multiple `localStorage` instances per origin.

Due to the synchronous nature of the Web Storage API, user agents implement
`localStorage` via a full in-memory cache.

TODO: Explain why we chose not to integrate `localStorage`.


## Stakeholder Feedback / Opposition

* Chrome : Positive, authoring this explainer
* [Implementor A] : Positive
* [Stakeholder B] : No signals
* [Implementor C] : Negative


## References & acknowledgements

Many thanks for valuable feedback and advice from:

* Andrew Sutherland
* Anne van Kesteren
* Asa Kusuma
* Ben Kelly
* Daniel Murphy
* Joshua Bell
* Marijn Kruisselbrink
* Staphany Park

This proposal is based on the
[Storage Buckets presentation at TPAC 2019](https://docs.google.com/presentation/d/143w70xtfqculs5x-bPgTOwu6GZO6uTOYeBjnN63pzs8/)
and the
[Storage Buckets meeting discussion](https://docs.google.com/document/d/1eBWhY91nUfdT2mys3GaNKX4fKPei79Wk-KlWHYffbAA/).
