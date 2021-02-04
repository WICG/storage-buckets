### StorageBuckets API Interface

#### The StorageBucketManager Interface
```
[
  Exposed=(Window,Worker),
  SecureContext
] interface StorageBucketManager {
    [NewObject] Promise<StorageBucket> open(DOMString name,
                                            optional StorageBucketOptions options = {});
    Promise<sequence<DOMString>> keys();
    Promise<undefined> delete(DOMString name);
};

dictionary StorageBucketOptions {
  DOMString? title = null;
  boolean persisted = false;
  StorageBucketDurability durability = "relaxed";
  unsigned long long? quota = null;
  DOMTimeStamp? expires = null;
};

enum StorageBucketDurability {
  "strict",
  "relaxed"
};

[
  Exposed=(Window,Worker),
  SecureContext
] interface StorageBucket {
  [Exposed=Window] Promise<boolean> persist();
  Promise<boolean> persisted();

  Promise<StorageEstimate> estimate();

  Promise<StorageBucketDurability> durability();

  Promise<undefined> setExpires(DOMTimeStamp expires);
  Promise<DOMTimeStamp> expires();
};
```

### navigator.storageBuckets
```
partial interface Navigator {
  [SecureContext, SameObject] readonly attribute StorageBucketManager storageBuckets;
};
```

### Integration with IndexedDB
```
partial interface StorageBucket {
  [SameObject] readonly attribute IDBFactory indexedDB;
};
```

### Integration with Cache Storage
```
partial interface StorageBucket {
  [SameObject] readonly attribute CacheStorage caches;
};
```

### Integration with Web Locks API
```
partial interface StorageBucket {
  [SameObject] readonly attribute LockManager locks;
};
```

### Integration with File System Access
```
partial interface StorageBucket {
  Promise<FileSystemDirectoryHandle> getDirectory();
};
```

### Integration with File API
```
partial interface StorageBucket {
  [NewObject] Promise<Blob> createBlob(optional sequence<BlobPart> blobParts,
                                       optional BlobPropertyBag options = {});
  [NewObject] Promise<File> createFile(sequence<BlobPart> fileBits,
                                       USVString fileName,
                                       optional FilePropertyBag options = {});
};
```

### Integration with Service Worker
`StorageBucket` exposes a subset of methods from [ServiceWorkerContainer](https://w3c.github.io/ServiceWorker/#serviceworkercontainer) and will include the following methods. 
- [register](https://w3c.github.io/ServiceWorker/#dom-serviceworkercontainer-register)
- [getRegistration](https://w3c.github.io/ServiceWorker/#dom-serviceworkercontainer-getregistration)
- [getRegistrations](https://w3c.github.io/ServiceWorker/#dom-serviceworkercontainer-getregistrations)


`ServiceWorkerContainerRegistration` will pull some members from `ServiceWorkerContainer` so it can be included in the `StorageBucket` interface, ultimately changing the Service Worker spec.
```
interface mixin ServiceWorkerContainerRegistration {
  [NewObject] Promise<ServiceWorkerRegistration> register(USVString scriptURL,
                                                          optional RegistrationOptions options = {});
  [NewObject] Promise<any> getRegistration(optional USVString clientURL = "");
  [NewObject] Promise<FrozenArray<ServiceWorkerRegistration>> getRegistrations();
};
```
```
interface StorageBucketServiceWorkerContainer {};

StorageBucketServiceWorkerContainer includes ServiceWorkerContainerRegistration;

partial interface StorageBucket {
  [SameObject] readonly attribute StorageBucketServiceWorkerContainer serviceWorker;
};
```
