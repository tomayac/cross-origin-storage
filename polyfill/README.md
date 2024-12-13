# Cross-Origin Storage Polyfill

The **Cross-Origin Storage (COS) Polyfill** provides a JavaScript implementation of the proposed [Cross-Origin Storage (COS) API](https://github.com/tomayac/cross-origin-storage), enabling web applications to store and retrieve large files securely across origins with explicit user consent.

> [!CAUTION]
> This polyfill doesn't actually work as the final API would. It still stores files redundantly as it can't overcome storage partitioning. It aims at emulating the behavior of the `navigator.crossOriginStorage.requestFileHandle()` function.

## Overview

The polyfill simulates the COS API to:

1. **Store files across origins** with SHA-256 hashes for unique identification and user-friendly names for permission management.
2. **Retrieve files from storage**, enabling applications to share large resources like AI models, Wasm modules, or database files.
3. **Ensure user privacy and security**, with explicit permission prompts for file storage and access.

## Key features

- **Cross-origin file storage and retrieval** with secure hashing for consistency.
- **Human-readable names** for user-friendly file management.
- **Browser-based caching** using the Cache API for efficient storage.
- **User consent** mechanisms via prompts for file access and storage.

## How it works

### Architecture

The polyfill consists of several components:

1. **Polyfill iframe (`iframe.html`)**
   Acts as a secure intermediary for cross-origin file management, responding to `postMessage()` requests from client applications.

1. **Polyfill library (`cos-polyfill.js`)**
   Provides a `navigator.crossOriginStorage.*` API, exposing methods to request file handles, store files, and retrieve files.

1. **Utility Functions (`util.js`)**
   Includes helper functions like `getBlobHash()` to compute SHA-256 hashes for files.

### Flow of operations

#### Storing a file

1. A client invokes `navigator.crossOriginStorage.requestFileHandle()` with a file's hash, a human-readable name, and `create: true`.
1. The polyfill prompts the user to grant storage permission.
1. Upon approval, the polyfill stores the file in the browser's Cache API, keyed by its hash.

#### Retrieving a file

1. A client invokes `navigator.crossOriginStorage.requestFileHandle()` with a file's hash and name.
1. The polyfill prompts the user to grant access permission.
1. If the file exists in the cache, it is returned as a Blob. If not, the client may fetch it from the network.

### Example usage

#### Storing a file

```js
const hash = 'SHA-256:8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4';
const name = 'Large AI Model';

try {
  const handle = await navigator.crossOriginStorage.requestFileHandle(hash, {
    name,
    create: true,
  });

  const writableStream = await handle.createWritable();
  const fileBlob = await fetch('https://example.com/large-ai-model.bin').then(
    (res) => res.blob()
  );
  await writableStream.write(fileBlob);
  await writableStream.close();

  console.log(`Stored file: ${name}`);
} catch (err) {
  console.error(err.name, err.message);
}
```

#### Retrieving a file

```javascript
const hash = 'SHA-256:8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4';
const name = 'Large AI Model';

try {
  const handle = await navigator.crossOriginStorage.requestFileHandle(hash, {
    name,
  });

  const fileBlob = await handle.getFile();
  console.log(`Retrieved file: ${name}`, fileBlob);
} catch (err) {
  console.error(err.name, err.message);
}
```

## Demo

Two testable clients are available:

- [COS Client 1](https://cos-client1.glitch.me/)
- [COS Client 2](https://cos-client2.glitch.me/)

Both clients embed the [COS polyfill iframe](https://tomayac.github.io/cross-origin-storage/polyfill/iframe.html).