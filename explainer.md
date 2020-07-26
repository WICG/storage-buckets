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
- [Background](#background)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Use Cases](#use-cases)
- [API Design](#api-design)
- [Key Scenarios](#key-scenarios)
- [Considered alternatives](#considered-alternatives)
  - [Event naming](#event-naming)
  - [Event target](#event-target)
  - [Service worker availability](#service-worker-availability)
  - [Event dispatch logic](#event-dispatch-logic)
  - [Storage space reservation](#storage-space-reservation)
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

## Bucket creation

Applications are expected to deliberately set up buckets before using storage
APIs. In the simplest form, this looks as follows.

```javascript
// Create a bucket for emails that are synchronized with the server.
const inboxBucket = await navigator.storageBuckets.openOrCreate("inbox", {
  title: "Inbox",  // User agents may display this in storage management UI.
});

// Open the "messages" database inside the "inbox" bucket.
const inboxDb = await new Promise(resolve => {
  const request = inboxBucket.indexedDB.open("messages");
  request.onupgradeneeded = () => { /* migration code */ };
  request.onsuccess = () => resolve(request.result);
  request.onerror = () => reject(request.error);
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

// The "messages" inside the "drafts" bucket is different from the "messages"
// database in the "inbox" bucket. The two databases follow different storage
// policies.
const draftsDb = await new Promise(resolve => {
  const request = draftsBucket.indexedDB.open("messages");
  request.onupgradeneeded = () => { /* migration code */ };
  request.onsuccess = () => resolve(request.result);
  request.onerror = () => reject(request.error);
});
```

Creating a bucket with lower importance.
```javascript
const cacheBucket = await navigator.storageBuckets.openOrCreate(
  "cache",
  {
    durability: "relaxed",
    persisted: false,
    title: "Recently Seen Stuff",
  });
```

## The default bucket

The following existing APIs operate on the `default` bucket. These APIs create
the `default` bucket on-demand.

* `WindowOrWorkerGlobalScope.indexedDB` in
  [IndexedDB](https://w3c.github.io/IndexedDB/#factory-interface)
* `WindowOrWorkerGlobalScope.caches` in
  [ServiceWorker](https://w3c.github.io/ServiceWorker/#self-caches)
* `NavigatorStorage.storage` in
  [the Storage standard](https://storage.spec.whatwg.org/#api)
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

### Opening Buckets
```javascript
const cache = await draftsBucket.caches.open("images");

const idb = await new Promise((resolve) => {
  const req = draftsBucket.indexedDB.open("drafts-folder", 1);
  req.onupgradeneeded = { /*…*/ };
  req.onsuccess = (evt) => { resolve(evt.target.result); };
});
```

### Reserving Quota
```javascript
const autosaveBucket = await navigator.storageBuckets.open(
  "autosave",
  {
    title: "Autosaved Form Data",
    durability: "strict",
  });
await autosaveBucket.reserve(20 * 1024 * 1024);  // 20 MB
```

### Estimate Usage
```javascript
const draftsUsage = await draftsBucket.estimate();
```

### Delete a Bucket
```javascript
await navigator.storageBuckets.delete(draftsBucket.name);
```

### Buckets in Service Workers
```javascript
const appBucket = await navigator.storageBuckets.open("app");
const reg = await appBucket.serviceWorker.register("sw.js");
// OR
const reg = await navigator.serviceWorker.register(
    "sw.js", { bucket: appBucket });
// THEN MAYBE
Clear-Site-Data: "storage:app"
```

###
[For each related element of the proposed solution - be it an additional JS
method, a new object, a new element, a new concept etc., create a section
which briefly describes it.]


```javascript
// Provide example code - not IDL - demonstrating the design of the feature.

// If this API can be used on its own to address a user need,
// link it back to one of the scenarios in the goals section.

// If you need to show how to get the feature set up
// (initialized, or using permissions, etc.), include that too.
```

[Where necessary, provide links to longer explanations of the relevant
pre-existing concepts and API. If there is no suitable external
documentation, you might like to provide supplementary information as an
appendix in this document, and provide an internal link where appropriate.]

[If this is already specced, link to the relevant section of the spec.]

[If spec work is in progress, link to the PR or draft of the spec.]


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

TODO: Discussion around allowable characters.


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

### [Alternative 1]

[Describe an alternative which was considered, and why you decided against it.]

### Alternative name for bucket `title`

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
