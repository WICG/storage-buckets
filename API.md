### StorageBuckets API Interface

### The StorageBucketManager Interface
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
  unsigned long? quota = null,
  DOMTimeStamp? expires = null,
}

enum StorageBucketDurability {
  "strict",
  "relaxed"
}

dictionary StorageBucket {
  [Exposed=Window] Promise<boolean> persist();
  Promise<boolean> persisted();

  Promise<StorageEstimate> estimate();

  Promise<StorageBucketDurability> durability();

  Promise<undefined> setExpires(DOMTimeStamp expires);
  Promise<DOMTimeStamp> expires();
}
```

## navigator.storageBuckets
```
partial interface Navigator {
  [SecureContext, SameObject] readonly attribute StorageBucketManager storageBuckets;
}
```

## Storage Manager
[SecureContext] partial interface StorageBucket {
  estimate();
}

## Indexed DB
```
[SameObject] partial interface StorageBucket {
  readonly attribute IDBFactory indexedDB;
}
```

## Cache Storage
```
[SecureContext, SameObject] partial interface StorageBucket {
  readonly attribute CacheStorage caches;
}
```

## File System Access
```
[SecureContext] partial interface StorageBucket {
  promise<FileSystemDirectoryHandle> getDirectory();
}
```

## Service Worker
```
[SecureContext] partial interface StorageBucket {
  readonly attribute ServiceWorkerContainer serviceWorker;
}
```

## File API
```
partial interface StorageBucket {
  Promise<Blob> createBlob(BlobPart, Blob);
  Promise<File> createFile();
}
```
