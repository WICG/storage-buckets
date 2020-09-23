# Security and Privacy Questionnaire 

[Security and Privacy questionnaire](https://www.w3.org/TR/security-privacy-questionnaire/)
responses for the Storage Buckets API

### 2.1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?
No new information will be exposed from this feature. 

### 2.2. Is this specification exposing the minimum amount of information necessary to power the feature?
Yes, no new outside information is exposed. The new information stored by this feature follows the same-origin principle,
and is the minimum amount of information needed to meet the use cases in the explainer. 

### 2.3. How does this specification deal with personal information or personally-identifiable information or information derived thereof?
This feature does not expose new personally-identifiable information to sites.
Sites may use this feature to store and retrieve personally-identifiable information that they already have. 

### 2.4. How does this specification deal with sensitive information?
This feature does not expose any new sensitive information. 

### 2.5. Does this specification introduce new state for an origin that persists across browsing sessions?
Yes, this feature adds the ability to create multiple storage buckets for an origin, in addition to the currently-specified default bucket.
Buckets have names and storage policies.

This feature does not add a new storage capability to the Web platform. All of its metadata could be stored in one of the existing storage APIs. 

### 2.6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?
No underlying platform information is exposed directly.

An origin may observe bucket eviction and infer that the user's device is experiencing storage pressure. This information is already exposed in today's one-bucket-per-origin world, as an origin may observe that its session cookies are intact, but its storage has been evicted.

### 2.7. Does this specification allow an origin access to sensors on a user’s device
No.

### 2.8. What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.
This feature introduces multiple storage buckets per origin, where each bucket has some metadata (name, description) and storage policies (expiration, durability, persistence) associated with it. All the newly introduced persistent data follows the same-origin principle.

Each storage bucket can store data associated with one of the following storage APIs.

* [IndexedDB](https://w3c.github.io/IndexedDB/)
* [CacheStorage](https://w3c.github.io/ServiceWorker/#cachestorage)
* Origin-Private File System in the [File System Access API](https://wicg.github.io/file-system-access/#sandboxed-filesystem)

Each storage bucket supports quota accounting exposed by the [StorageManager](https://storage.spec.whatwg.org/#storagemanager) API.

### 2.9. Does this specification enable new script execution/loading mechanisms?
No.

### 2.10. Does this specification allow an origin to access other devices?
No. 

### 2.11. Does this specification allow an origin some measure of control over a user agent’s native UI?
No.

### 2.12. What temporary identifiers might this specification create or expose to the web?
None.

### 2.13. How does this specification distinguish between behavior in first-party and third-party contexts? 
We are currently focusing on functionality for first-party contexts, and are not proposing any distinction between first-party and third-party contexts.

However, our proposal allows for follow-up work that may restrict access in third-party contexts. For example, a user agent may decide that third-party contexts can only access a special bucket, which would be set as the default bucket, and `navigator.storageBuckets.openOrCreate()` throws `SecurityError`.

### 2.14. How does this specification work in the context of a user agent’s Private Browsing or "incognito" mode?
User agents are expected to maintain separate state for Private Browsing sessions.

More concretely, the collection of buckets exposed to origins in a Private Browsing session should be separate from the collection exposed in a normal browsing session.

At the time of this writing, major browsers have RAM-backed implementations of storage APIs such as [IndexedDB](https://w3c.github.io/IndexedDB/) and [CacheStorage](https://w3c.github.io/ServiceWorker/#cachestorage) for Private Browsing sessions. We expect that these browsers will have RAM-backed implementations of Storage Buckets as well.

### 2.15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?
This explainer targets [the Storage specification](https://storage.spec.whatwg.org/). We expect to add a "Security Considerations" section discussing quota accounting (the `estimate()` API) and opaque responses, and a "Privacy Considerations" section discussing bucket eviction.

### 2.16. Does this specification allow downgrading default security characteristics?
No.

### 2.17. What should this questionnaire have asked?
