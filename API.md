### StorageBuckets API Interface

#### The StorageBucketManager Interface
```
[
  Exposed=(Window,Worker),
  SecureContext
] interface StorageBucketManager {
    Promise<StorageBucket> openOrCreate(USVString name,
                                        optional StorageBucketOptions options = {});
    Promise<sequence<USVString>> keys();
    Promise<undefined> delete(USVString name);
};

dictionary StorageBucketOptions {
  USVString? title = null,
  boolean persisted = false,
  StorageBucketDurability durability = "relaxed",
  unsigned long long? quota = null,
  DOMTimeStamp? expires = null,
}

enum StorageBucketDurability {
  "strict",
  "relaxed"
}

[SecureContext] interface StorageBucket {
  [Exposed=Window] Promise<boolean> persist();
  Promise<boolean> persisted();

  Promise<StorageEstimate> estimate();

  Promise<StorageBucketDurability> durability();

  Promise<undefined> setExpires(DOMTimeStamp expires);
  Promise<DOMTimeStamp> expires();
}
```

### navigator.storageBuckets
```
partial interface Navigator {
  [SecureContext, SameObject] readonly attribute StorageBucketManager storageBuckets;
}
```

### Indexed DB
```
partial interface StorageBucket {
  [SameObject] readonly attribute IDBFactory indexedDB;
}
```

### Cache Storage
```
partial interface StorageBucket {
  [SameObject] readonly attribute CacheStorage caches;
}
```

### File System Access
```
partial interface StorageBucket {
  promise<FileSystemDirectoryHandle> getDirectory();
}
```

### Service Worker
```
partial interface StorageBucket {
  [SameObject] readonly attribute ServiceWorkerContainer serviceWorker;
}
```

### File API
```
partial interface StorageBucket {
  Promise<Blob> createBlob(optional sequence<BlobPart> blobParts,
                           optional BlobPropertyBag options = {});
  Promise<File> createFile(sequence<BlobPart> fileBits,
                           USVString fileName,
                           optional FilePropertyBag options = {});
}
```
