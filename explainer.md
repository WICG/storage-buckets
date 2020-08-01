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
- [Storage policies](#storage-policies)
- [Getting a bucket's quota usage](#getting-a-buckets-quota-usage)
- [Reserving quota for a bucket](#reserving-quota-for-a-bucket)
- [The default bucket](#the-default-bucket)
- [Storage buckets and service workers](#storage-buckets-and-service-workers)
- [Storage buckets and the Clear-Site-Data](#storage-buckets-and-the-clear-site-data)
- [Key Scenarios](#key-scenarios)
  - [Scenario 1](#scenario-1)
  - [Scenario 2](#scenario-2)
- [Detailed design discussion](#detailed-design-discussion)
  - [Bucket names](#bucket-names)
  - [Bucket titles](#bucket-titles)
  - [Storage policy naming](#storage-policy-naming)
  - [[Tricky design choice 2]](#tricky-design-choice-2)
- [Considered alternatives](#considered-alternatives)
  - [Bucket designations for each storage API](#bucket-designations-for-each-storage-api)
  - [Alternative name for the bucket `title` property](#alternative-name-for-the-bucket-title-property)
  - [Language maps for bucket titles](#language-maps-for-bucket-titles)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


## Introduction

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
- Allow web applications to evict partitions of data
- Allow web developers to specify eviction ordering
- Allow web applications to easily evict service workers without clearing data for the entire domain
- Allow web developers to reserve quota
- Allow web developers to express performance, durability and other trade-off decisions on partitions of data
- Ensure privacy across accounts on shared devices
- Allow users to have control over which data to evict, and prevent important data from being deleted

## Non-goals

[If there are “adjacent” goals which may appear to be in scope but aren’t,
enumerate them here. This section may be fleshed out as your design
progresses and you encounter necessary technical and other trade-offs.]

## Use Cases
[Motivating use cases, or scenarios]

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
  title: "Inbox",  // User agents may display this in storage management UI.
});
```

Buckets can be assigned different storage policies at creation time. The example
below demonstrates some policies appropriate for data that is not (yet)
synchronized with a server. The policies introduced by this proposal will be
described in future sections.

```javascript
const draftsBucket = await navigator.storageBuckets.openOrCreate("drafts", {
  durability: "strict",
  persisted: true,
  title: "Drafts",
});
```


## Getting the storage policies associated with a bucket

The storage policies passed to `openOrCreate()` are advisory. User agents may
create buckets whose policies don't match the requests. In most cases, the
deviations only result in different performance characteristics. Applications
can check a bucket's policies and take appropriate action when a vital policy
does not match the desired value.

```javascript
const draftsBucket = await navigator.storageBuckets.openOrCreate("drafts", {
  durability: "strict",
  persisted: true,
  title: "Drafts",
});

if (draftsBucket.persisted !== "strict") {
  displayWarningButterBar("Your drafts may be lost if the computer loses power!");
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


## Storage policy: Persistence

This explainer introduces parameters for the following policies.

* `persisted` was chosen for consistency with
[StorageManager.persisted()](https://storage.spec.whatwg.org/#dom-storagemanager-persisted)
in the Storage specification. The true / false values are also consistent with
the definitions in the Storage specification.


## Storage policy: Durability

`durability` was chosen for consistency with
[IDBTransaction.durability](https://w3c.github.io/IndexedDB/#dom-idbtransaction-durability)
in IndexedDB. The values `"strict"` and `"relaxed"` have the same significance
as the corresponding
[IndexedDB transaction hints](https://w3c.github.io/IndexedDB/#transaction-durability-hint).


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
  durability: "strict",
  persist: false,
  title: "",
});
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

[If there are a suite of interacting APIs, show how they work together to
solve the key scenarios described.]

### Scenario 1

[Description of the end-user scenario]

```javascript
// Sample code demonstrating how to use these APIs to address that scenario.
```

### Scenario 2

[etc.]

## Detailed design discussion

### Bucket names

TODO: `/` intended for tree display in UI.

TODO: Discussion around allowable characters. The set should make it easy to
specify buckets in Clear-Site-Data.


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
   failure, but this is not generally guaranteed.

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

TODO: Explain that the oher options were ignored as a result of trading off
better control (which might result in better performance) against presenting
developers with a simple model (writing guidance for more options would be
significantly more complex) and against cross-system consistency. We consider
that the performance difference between application-level buffers OS-level
buffers is not significant (comparable to IPC in a multi-process browser). The
ability to flush to storage device buffers is not available on all operating
systems.

TODO: Explain that `relaxed` is the default because applications need to design
explicitly for being able to recover from power failures. The durability
guarantees apply to individual writes, but applications need to handle
inconsistencies across storage APIs (Cache Storage and IndexedDB). We don't
offer two-phase commit across storage APIs.


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
  durability: "strict",
  persisted: true,
  title: "Drafts",
});

await inboxBucket.close();
await draftsBucket.close();

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
  title: "Inbox",
});

// Fails if the "inbox" bucket does not already exist.
const inboxBucket = await navigator.storageBuckets.open("inbox", {
  title: "Inbox",
});

// Fails if the "inbox" bucket already exists.
const inboxBucket = await navigator.storageBuckets.create("inbox", {
  title: "Inbox",
});
```

Alternatively, we could have settled for one `open()` method with options, as
shown below.

```javascript
// By default, creates the "inbox" bucket if does not already exist.
const inboxBucket = await navigator.storageBuckets.open("inbox", {
  title: "Inbox",
});

// Fails if the "inbox" bucket does not already exist.
const inboxBucket = await navigator.storageBuckets.open("inbox", {
  title: "Inbox", failIfNotExist: true,
});

// Fails if the "inbox" bucket already exists.
const inboxBucket = await navigator.storageBuckets.open("inbox", {
  title: "Inbox", failIfExists: true,
});
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

const draftBlob = new Blob(
    ["Message text."], { type: "text/plain", bucket: "drafts" });
const draftFile = new File(
    ["Attachment data"], "attachment.txt",
    { type: "text/plain", lastModified: Date.now(), bucket: "drafts" });

const inboxTestDir = await self.getOriginPrivateDirectory({ bucket: "inbox" });

const inboxRegistration = await navigator.serviceWorker.register(
    "/inbox-sw.js", { scope: "/inbox", bucket: "inbox" });
```

TODO: Explain why this is worse than the main decision.


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

```js
const draftsBucket = await navigator.storageBuckets.openOrCreate("drafts", {
  durability: "strict",
  persisted: true,
  title: {
    en: "Drafts",
    es: "Borradoers",
    jp: "下書き",
  }
});
```

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
