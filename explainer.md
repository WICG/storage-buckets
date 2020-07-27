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

if (draftsBucket.durability !== "strict") {
  displayWarningButterBar("Your drafts may be deleted while you're offline!");
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


## Storage policies

This explainer introduces parameters for the following policies.

* `persisted` was chosen for consistency with
[StorageManager.persisted()](https://storage.spec.whatwg.org/#dom-storagemanager-persisted)
in the Storage specification. The true / false values are also consistent with
the definitions in the Storage specification.

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
const autosaveBucket = await navigator.storageBuckets.open(
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

[Talk through the tradeoffs in coming to the specific design point you want
to make.]

```javascript
// Illustrated with example code.
```

[This may be an open question, in which case you should link to any active
discussion threads.]


### [Tricky design choice 2]

[etc.]


## Considered alternatives

[This should include as many alternatives as you can, from high level
architectural decisions down to alternative naming choices.]

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
