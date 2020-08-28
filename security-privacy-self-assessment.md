# Security and Privacy Questionnaire 

[Security and Privacy questionnaire](https://www.w3.org/TR/security-privacy-questionnaire/)
responses for the Storage Buckets API

### 2.1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?
No new information will be exposed from this feature. 

### 2.2. Is this specification exposing the minimum amount of information necessary to power the feature?
Yes, this feature will not be adding more exposure of information. 

### 2.3. How does this specification deal with personal information or personally-identifiable information or information derived thereof?
This feature does not deal with personally-identifiable information therefore will not be exposing any personally-identifiable information. 

### 2.4. How does this specification deal with sensitive information?
This feature does not expose any new sensitive information. 

### 2.5. Does this specification introduce new state for an origin that persists across browsing sessions?
No.

### 2.6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?
None. 

### 2.7. Does this specification allow an origin access to sensors on a user’s device
No.

### 2.8. What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.
TODO

### 2.9. Does this specification enable new script execution/loading mechanisms?
No.

### 2.10. Does this specification allow an origin to access other devices?
No. 

### 2.11. Does this specification allow an origin some measure of control over a user agent’s native UI?
No.

### 2.12. What temporary identifiers might this specification create or expose to the web?
None.

### 2.13. How does this specification distinguish between behavior in first-party and third-party contexts? 
No distinction. This feature is inaccessible by third-party contexts. 

### 2.14. How does this specification work in the context of a user agent’s Private Browsing or "incognito" mode?
This feature is an entry point for established storage APIs such as [IndexedDB](https://w3c.github.io/IndexedDB/) and [CacheStorage](https://w3c.github.io/ServiceWorker/#cachestorage). The behavior under Private Browser or "incognito" will remain unchanged. 

### 2.15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?
TODO

### 2.16. Does this specification allow downgrading default security characteristics?
No.

### 2.17. What should this questionnaire have asked?
