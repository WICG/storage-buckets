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

Each storage bucket can store data associated with established storage APIs such
as [IndexedDB](https://w3c.github.io/IndexedDB/) and
[CacheStorage](https://w3c.github.io/ServiceWorker/#cachestorage).

[The “executive summary” or “abstract”. Explain in a few sentences what the
goals of the project are, and a brief overview of how the solution works.
This should be no more than 1-2 paragraphs.]


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

## API Design

### Create a Bucket

Simple bucket creation.
```javascript
const draftBucket = await navigator.storageBuckets.openOrCreate("mail/drafts");
```

Here we create a bucket of high importance where durablity, encryption and reserved quota is specified.
```javascript
const draftBucket = await navigator.storageBuckets.openOrCreate("mail/drafts", {
  title: "Email Drafts",
  durability: "strict",
  persist: true,
  encrypt: "s3cr3t",
  maxQuota: 128 * 1024,  // Explicitly reserve 128 KB of quota.
  ...
});
```

Creating a bucket with lower importance.
```javascript
const cacheBucket = await navigator.storageBuckets.openOrCreate(
  "cache",
  {
    title: "Recently Seen Stuff",
    durability: "relaxed",
    persist: false,
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

### [Tricky design choice #1]

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

### [Alternative 2]

[etc.]


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
