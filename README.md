# Explainer for the Cross-Origin Storage (COS) API

<img src="https://raw.githubusercontent.com/tomayac/cross-origin-storage/refs/heads/main/logo-cos.svg" alt="Cross-Origin Storage (COS) logo, consisting of a folder icon with a crossing person." width="100">

This proposal outlines the design of the **Cross-Origin Storage (COS)** API, which allows web applications to store and retrieve files across different origins with explicit user consent. Using concepts introduced in **File System Living Standard** defined by the WHATWG, the COS API facilitates secure cross-origin file storage and retrieval for large files, such as AI models, SQLite databases, offline storage archives, and WebAssembly (Wasm) modules. Taking inspiration from **Cache Digests for HTTP/2**, the API uses file hashes for integrity, while human-readable descriptions allow for permission management.

## Authors

- [Thomas Steiner](mailto:tomac@google.com), Google Chrome
- [Christian Liebel](mailto:christian@liebel.org), Thinktecture AG

## Participate

- [Issues](https://github.com/tomayac/cross-origin-storage/issues)
- [PRs](https://github.com/tomayac/cross-origin-storage/pulls)

## Introduction

The **Cross-Origin Storage (COS)** API provides a cross-origin file storage and retrieval mechanism for web applications. It allows applications to store and access large files, such as AI models, SQLite databases, offline storage archives, and Wasm modules across different origins securely and with user consent. Taking inspiration from **Cache Digests for HTTP/2**, files are identified by their hashes to ensure integrity, and human-readable descriptions need to be assigned to files for permission management. The API uses concepts like `FileSystemFileHandle` from the **File System Living Standard** with a focus on cross-origin usage. Here is an example that shows the basic flow for retrieving a file from COS:

```js
// The hash of the file we want to access.
const hash = {
  algorithm: 'SHA-256',
  value: '8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4',
};
// The human-readable file description.
const description = 'Large AI Model';

// This triggers a permission prompt:
// example.com wants to check if the file "Large AI Model" is stored by your browser.
// [Allow this time] [Allow on every visit] [Don't allow]
try {
  const handle = await navigator.crossOriginStorage.requestFileHandle(hash, {
    description,
  });
  // The file exists in Cross-Origin Storage.
  const fileBlob = await handle.getFile();
  // Do something with the blob.
  console.log('Retrieved', description, fileBlob);
} catch (err) {
  if (err.name === 'NotAllowedError') {
    console.log('The user did not grant permission to access the file.');
    return;
  }
  // `NotFoundError`, the file wasn't found in COS.
  console.error(err.name, err.message);
}
```

## Risk awareness

> [!CAUTION]
> The authors acknowledge that storage is usually isolated by origin to safeguard user security and privacy. Storing large files like AI models or SQLite databases separately for each origin, as required by new [use cases](#use-cases), presents a different challenge. For instance, if both `example.com` and `example.org` each require the same 8&nbsp;GB AI model, this would result in a total of 16&nbsp;GB downloaded data and a total allocation of 16&nbsp;GB on the user's device. The present proposal centers on effective mechanisms that uphold protection standards while addressing the inefficiencies of duplicated download and storage.

## Goals

COS aims to:

- Provide a cross-origin storage mechanism for web applications to store and retrieve large files like AI models, SQLite databases, offline storage archives (for example, complete website archives at the scale of Wikipedia), and Wasm modules.
- Ensure security and user control with explicit consent before accessing or storing files.
- Use SHA-256 hashes (see [Appendix B](#appendix-b-blob-hash-with-the-web-crypto-api)) for file identification, guaranteeing data integrity and consistency.
- Require developers to assign human-readable descriptions to files for permission management.

## Non-goals

COS does _not_ aim to:

- Replace existing storage solutions such as the **Origin Private File System**, the **Cache API**, **IndexedDB**, or **Web Storage**.
- Replace content delivery networks (CDNs). The required prompting is expected to deter websites from using the COS API unless there's a clear benefit to cross-origin file access, such as potentially utilizing a cached version.
- Store popular JavaScript libraries like jQuery. (See the [FAQ](#appendix-c-faq).)
- Provide backend or cloud storage solutions.
- Allow cross-origin file access _without_ explicit user consent.

> [!IMPORTANT]
> COS has distinct objectives from the [Shared Storage API](https://github.com/WICG/shared-storage) proposal, which serves as common key/value storage infrastructure for privacy-preserving cross-site use cases.
>
> It is also distinct from the [Related Website Partition API](https://github.com/explainers-by-googlers/related-website-partition-api) proposal, which allows third-party embeds to request access to a storage partition that is accessible across sites in a single [Related Website Set](https://wicg.github.io/first-party-sets/).

## User research

Feedback from developers working with large AI models, SQLite databases, offline storage archives, and Wasm modules has highlighted the need for an efficient way to store and retrieve such large files across web applications on different origins. These developers are looking for a standardized solution that allows files to be stored once and accessed by multiple applications, without needing to download and store the files redundantly. COS ensures this is possible while maintaining privacy and security via user consent.

## Use cases

### Use case 1: Large AI models

Developers working with large AI models can store these models once and access them across multiple web applications. By using the COS API, models can be stored under their hashes and retrieved with user consent, minimizing repeated downloads and storage, ensuring file integrity. An example is Google's [Gemma 2](https://huggingface.co/google/gemma-2-2b/tree/main) model [`g-2b-it-gpu-int4.bin`](https://storage.googleapis.com/jmstore/kaggleweb/grader/g-2b-it-gpu-int4.bin') (1.35&nbsp;GB). Another example is Google's [Gemma 1.1 7B](https://huggingface.co/google/gemma-1.1-7b-it) model `gemma-1.1-7b-it` (8.60&nbsp;GB), which can be [run in the browser](https://research.google/blog/unlocking-7b-language-models-in-your-browser-a-deep-dive-with-google-ai-edges-mediapipe/). Yet another example is the [`Llama-3-70B-Instruct-q3f16_1-MLC`](https://huggingface.co/mlc-ai/Llama-3-70B-Instruct-q3f16_1-MLC/tree/main) model (33&nbsp;GB), which [likewise runs in the browser](https://chat.webllm.ai/) (choose the "Llama 3.1 70B Instruct" model in the picker).

### Use case 2: Large database files and offline storage archives

Web applications may depend on large SQLite databases, for example, for geodata as provided by Geocode Earth [`whosonfirst-data-admin-latest.db.bz2`](https://geocode.earth/data/whosonfirst/combined/) (8.00&nbsp;GB). Another use case involves large archives, for example, [ZIM files](https://wiki.openzim.org/wiki/ZIM_file_format) like [`wikipedia_en_all_maxi_2024-01.zim`](https://library.kiwix.org/#lang=eng&category=wikipedia) (109.89&nbsp;GB) as used by PWAs like [Kiwix](https://pwa.kiwix.org/www/index.html). Storing such files once with the COS API has the advantage that multiple web apps can share the same files.

### Use case 3: Large Wasm modules

Web applications that utilize large Wasm modules can store these modules using COS and access them across different origins. This enables efficient sharing of files between applications, reducing redundant downloading and improving performance. Google's Flutter framework alone has four files that are used by more than 1,000 hosts each day making more than two million daily requests in total.

| Request (`https://gstatic.com/flutter-canvaskit/`)                                                                                                                           | Size   | Hosts | Requests |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ | ----- | -------- |
| [`36335019a8eab588c3c2ea783c618d90505be233/chromium/canvaskit.wasm`](https://gstatic.com/flutter-canvaskit/36335019a8eab588c3c2ea783c618d90505be233/chromium/canvaskit.wasm) | 5.1 MB | 1,938 | 596,900  |
| [`a18df97ca57a249df5d8d68cd0820600223ce262/chromium/canvaskit.wasm`](https://gstatic.com/flutter-canvaskit/a18df97ca57a249df5d8d68cd0820600223ce262/chromium/canvaskit.wasm) | 5.1 MB | 1,586 | 579,380  |
| [`36335019a8eab588c3c2ea783c618d90505be233/canvaskit.wasm`](https://gstatic.com/flutter-canvaskit/36335019a8eab588c3c2ea783c618d90505be233/canvaskit.wasm)                   | 6.4 MB | 1,142 | 597,240  |
| [`a18df97ca57a249df5d8d68cd0820600223ce262/canvaskit.wasm`](https://gstatic.com/flutter-canvaskit/a18df97ca57a249df5d8d68cd0820600223ce262/canvaskit.wasm)                   | 6.4 MB | 1,014 | 288,800  |

(**Source:** Google-internal data from the Flutter team: "Flutter engine assets by unique hosts - one day - Dec 10, 2024".)

## Potential solution

### File Storage Process

The **COS** API will be available through the `navigator.crossOriginStorage` interface. Files will be stored and retrieved using their hashes, ensuring that each file is uniquely identified. A human-readable description needs to be provided for permission management of files.

#### Storing a file

1. Hash the contents of the file using SHA-256 (or an equivalent secure algorithm, see [Appendix B](#appendix-b-blob-hash-with-the-web-crypto-api)). The used hash algorithm is communicated as a valid [`HashAlgorithmIdentifier`](https://w3c.github.io/webcrypto/#dom-hashalgorithmidentifier).
1. Request a `FileSystemFileHandle` for the file, specifying the file's hash and a human-readable description. This will trigger a permission prompt if it's okay for the origin to check if the file is stored by the browser.
1. If a file with the hash already exists, return.
1. Else, store the file in the browser.

```js
/**
 * Example usage to store a file.
 */

// The hash of the file we want to access.
const hash = {
  algorithm: 'SHA-256',
  value: '8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4',
};
// The human-readable file description.
const description = 'Large AI model';

// This triggers a permission prompt:
// example.com wants to check if the file "Large AI Model" is stored by your browser.
// [Allow this time] [Allow on every visit] [Don't allow]
try {
  const handle = await navigator.crossOriginStorage.requestFileHandle(hash, {
    description,
  });
} catch (err) {
  if (err.name === 'NotFoundError') {
    // Load the file from the network.
    const fileBlob = await loadFileFromNetwork();
    try {
      const handle = await navigator.crossOriginStorage.requestFileHandle(hash, {
        ,
        // Set to `true` to create the file.
        create: true,
      });
      const writableStream = await handle.createWritable();
      await writableStream.write(fileBlob);
      await writableStream.close();
    } catch (err) {
      // The `write()` failed.
    }
    return;
  }
  // 'NotAllowedError', the user didn't grant access to the file.
  console.log('The user did not grant access to the file.');
}
```

#### Retrieving a file

1. Request a file handle using the file's hash and a human-readable description. This will trigger a permission prompt if it's okay for the origin to check if the file is stored by the browser.
1. Retrieve the file after the user has granted access.

```js
/**
 * Example usage to retrieve a file.
 */

// The hash of the file we want to access.
const hash = {
  algorithm: 'SHA-256',
  value: '8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4',
};
// The human-readable file description.
const description = 'Large AI model';

// This triggers a permission prompt:
// example.com wants to check if the file "Large AI Model" is stored by your browser.
// [Allow this time] [Allow on every visit] [Don't allow]
try {
  const handle = await navigator.crossOriginStorage.requestFileHandle(hash, {
    description,
  });
  // The file exists in COS.
  const fileBlob = await handle.getFile();
  console.log(`Retrieved file: ${description}`);
  // Return the file as a Blob.
  console.log(fileBlob);
} catch (err) {
  if (err.name === 'NotFoundError') {
    // Load the file from the network.
    const fileBlob = await loadFileFromNetwork();
    // Return the file as a Blob.
    console.log(fileBlob);
    return;
  }
  // 'NotAllowedError', the user didn't grant access to the file.
  console.log('The user did not grant access to the file.');
}
```

#### Storing and retrieving a file across unrelated sites

To illustrate the capabilities of the COS API, consider the following example where two unrelated sites want to interact with the same large language model. The first site stores the model in COS, while the second site retrieves it, each using different human-readable descriptions, one in English and one in Spanish.

##### Site A: Storing a large language model with an English description

On Site A, a web application stores a large language model in COS with a human-readable English description, "Large AI Model."

```js
// The hash of the file we want to access.
const hash = {
  algorithm: 'SHA-256',
  value: '8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4',
};
// The human-readable file description.
const description = 'Large AI model';

// This triggers a permission prompt:
// site-a.example.com wants to check if the file "Large AI Model" is stored by your browser.
// [Allow this time] [Allow on every visit] [Don't allow]
try {
  const handle = await navigator.crossOriginStorage.requestFileHandle(hash, {
    description,
  });

  // Use the file and return.
  // …
  return;
} catch (err) {
  if (err.name === 'NotFoundError') {
    // Load the file from the network.
    const fileBlob = await loadFileFromNetwork();
    // Compute the control hash using the method in Appendix B.
    const controlHash = await getBlobHash(fileBlob);
    // Check if control hash and known hash are the same.
    if (controlHash !== hash.value) {
      // Downloaded file and wanted file are different.
      // …
      return;
    }
    try {
      handle = await navigator.crossOriginStorage.requestFileHandle(hash, {
        description,
        create: true,
      });
      const writableStream = await handle.createWritable();
      await writableStream.write(fileBlob);
      await writableStream.close();

      console.log(`File stored: ${description}`);
    } catch (err) {
      // The `write()` failed.
    }
    return;
  }
  // 'NotAllowedError', the user didn't grant access to the file.
  console.log('The user did not grant access to the file.');
}
```

##### Site B: Retrieving the same model with a Spanish description

On Site B, entirely unrelated to Site A, a different web application happens to retrieve the same model from COS, but refers to it with a human-readable Spanish description, "Modelo de IA Grande."

```js
// The hash of the file we want to access.
const hash = {
  algorithm: 'SHA-256',
  value: '8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4',
};
// The human-readable file description.
const description = 'Modelo de IA Grande';

// This triggers a permission prompt:
// site-b.example.com wants to check if the file "Modelo de IA Grande" is stored by your browser.
// [Allow this time] [Allow on every visit] [Don't allow]
try {
  const handle = await navigator.crossOriginStorage.requestFileHandle(hash, {
    description,
  });
  const fileBlob = await handle.getFile();
  // This now logs the Spanish description, even if the file was stored with an English description by site A.
  console.log(`File retrieved with description: ${description}`);
  // Use the fileBlob as needed.
} catch (err) {
  if (err.name === 'NotFoundError') {
    // The file wasn't in COS.
    console.error(err.name, err.message);
    return;
  }
  // 'NotAllowedError', the user didn't grant access to the file.
  console.log('The user did not grant access to the file.');
}
```

##### Key points

- **Unrelated sites:** The two sites belong to different origins and do not share any context, ensuring the example demonstrates cross-origin capabilities.
- **Human-readable descriptions:** Each site assigns its own human-readable description, localized to the user's context. The description isn't shared across origins.
- **Cross-origin sharing:** Despite the different descriptions and origins, the file is securely identified by its hash, demonstrating the API's ability to facilitate cross-origin file storage and retrieval.

## Detailed design discussion

### User consent and permissions

The permission prompt must clearly display the file's description to ensure users understand what file they are being asked to store or retrieve. The goal is to strike a balance between providing sufficient technical details and maintaining user-friendly simplicity.

An **access permission** will be shown every time the `navigator.crossOriginStorage.requestFileHandle(hash, { description })` method is called without the `create` option set to `true`, which can happen to check for existence of the file and to obtain the handle to then get the actual file. The `description` will be part of the permission text. User agents can decide to allow this on every visit, or to explicitly ask upon each access attempt.

If the origin has stored the file before, the user agent can decide to not show a prompt.

If the user agent knows that the file exists, it can customize the permission prompt to differentiate the existence check and the access prompt:

- If the file doesn't exist:
  ```
  example.com wants to check if the file "large file" is stored by your browser.
  [Allow this time] [Allow on every visit] [Don't allow]
  ```
- If the file does exist:
  ```
   example.com wants to access the file "large file" stored by your browser.
   [Allow this time] [Allow on every visit] [Don't allow]
  ```

> [!IMPORTANT]
> The permission could mention other recent origins that have accessed the same file, but this may be misinterpreted by the user as information the current site may learn, which is never the case. Instead, the vision is that user agents would make information about origins that have (recently) accessed a file stored in COS available in special browser settings UI, as outlined in [Handling of eviction](#handling-of-eviction).

### Privacy

Since the file retrieved upon explicit user permission, there's no way for files stored in COS to become supercookies without raising the user's suspicion. Privacy-sensitive user agents can decide to prompt upon every retrieval operation, others can decide to only prompt once, and auto-allow from thereon. User agents can decide to not prompt if the present origin has stored the file before.

### Hashing

COS relies on the same hashing algorithm to be used for all files. It's not possible to mix hashing algorithms, since, without access to the original file, there's no way to verify if a hash generated with hashing _algorithm&nbsp;A_ corresponds to a hash generated with hashing _algorithm&nbsp;B_. The used hashing algorithm is encoded in the hash as a [`HashAlgorithmIdentifier`](https://w3c.github.io/webcrypto/#dom-hashalgorithmidentifier).

```js
const hash = {
  algorithm: 'SHA-256',
  value: '8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4',
};
```

The current hashing algorithm is [SHA-256](https://w3c.github.io/webcrypto/#alg-sha-256), implemented by the **Web Crypto API**. If hashing best practices should change, COS will reflect the [implementers' recommendation](https://w3c.github.io/webcrypto/#algorithm-recommendations-implementers) in the Web Crypto API.

### Human-readable descriptions

A valid human-readable description is a string that is not an empty string and that doesn't contain linebreaks or control characters.

## Open questions

### Concurrency

What should happen if two tabs depend on the same file, check COS, see the file is not in COS, and start downloading? Should this be handled smartly? How often does this happen in practice? In the worst case, the file gets downloaded twice, but would then still only be stored once.

### Minimum file size

Should there be a required minimum file size for a file to be eligible for COS? Most likely not, since it would be trivial to inflate the file size of non-qualifying files by adding space characters or comments. The assumption is that the required prompting would be scary enough for websites to only use COS for files where it really makes sense to have them available cross-origin, that is, where they could profit themselves from using a potentially already cached version rather than downloading their own version from the network.

### Handling of eviction

Browsers should likely treat files in COS under the same conditions as if they were `'persistent'` as per the [Storage Living Standard](https://storage.spec.whatwg.org/#persistence).

User agents are envisioned to offer browser settings UI for the user to see what files are stored in COS and what origins have least recently used each file. The user can then choose to delete files from COS using the UI.

Under critical storage pressure, user agents could offer a manual dialog that invites the user to manually free up storage.

When the user clears site data, all usage information associated with the origin should be removed from files in COS. If a file in COS, after the removal of usage information, is deemed unused, the user agent may delete it from COS.

### Manual COS management

If a user already has manually downloaded a file like a large AI model, should the browser offer a way to let the user put the file in COS? This could just be an affordance provided by the user agent.

## Considered alternatives

### Storing files without hashing

Storing files by names rather than using hashes would risk name collisions, especially in a cross-origin environment. The use of hashes guarantees unique identification of each file, ensuring that the contents are consistently recognized and retrieved.

### Manually accessing files from harddisk

Different origins can manually open the same file on disk, either using the File System Access API's `showOpenFilePicker()` method or using the classic `<input type="file">` approach. This requires the file to be stored once, and access to the file can then be shared as explained in [Cache AI models in the browser](https://developer.chrome.com/docs/ai/cache-models#special_case_use_a_model_on_a_hard_disk). While this works, it's manual and error-prone, as it requires the user to know what file to choose from their harddisk in the file picker.

### Integrating cross-origin storage in the `fetch()` API

On the server, cross-origin isolation isn't really a problem. At the same time, server runtimes like Node.js, Bun, or Deno implement `fetch()` as well. To avoid fragmentation and to keep the present `fetch()` API simple, it probably doesn't make sense to add COS to `fetch()`. Since `fetch()` is URL-based, this would also not solve the case where the same file is stored at different locations.

### Integrating cross-origin storage in the Cache API

The Cache API is fundamentally modeled around the concepts of `Request` or URL strings, and `Response`, for example, `Cache.match()` or `Cache.put()`. In contrast, what makes COS unique is that it uses file hashes as the keys to files to avoid duplicates.

### Solving the problem only for AI models

AI models are admittedly the biggest motivation for working on COS, so one alternative would be to solve the problem exclusively for AI models, for example, by offering a storage mechanism on the `self.ai.*` namespace that Chrome is experimenting with in the context of built-in AI APIs like the [Prompt API](https://github.com/webmachinelearning/prompt-api) proposal. Two questions arise in the context: First, how would it be enforced that files are really AI models? Second, `self.ai.*` is explicitly focused on built-in AI APIs where the model is provided by the browser and not by the developer. Given this background, this approach doesn't seem like a great fit, and, maybe more importantly, the non-AI [use cases](#use-cases) are well worth solving, too.

## Stakeholder feedback / opposition

- **Web Developers**: Positive feedback for enabling sharing large files without repeated downloads and storage, particularly in the context of huge AI models, SQLite databases, offline storage archives, and large Wasm modules.

## References

- [File System Living Standard](https://fs.spec.whatwg.org/)
- [Web Cryptography API](https://w3c.github.io/webcrypto/)
- [Storage Living Standard ](https://storage.spec.whatwg.org/)
- [Cache Digests for HTTP/2](https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-cache-digest)

## Acknowledgements

Many thanks for valuable feedback from:

- **François Beaufort**, Google Chrome

Many thanks for valuable inspiration or ideas from:

- **Kenji Baheux**, Google Chrome
- **Kevin Moore**, Google Chrome

## Appendices

### Appendix A: Full IDL

```webidl
interface mixin NavigatorCrossOriginStorage {
  [SameObject, SecureContext] readonly attribute CrossOriginStorageManager crossOriginStorage;
};
Navigator includes NavigatorCrossOriginStorage;
WorkerNavigator includes NavigatorCrossOriginStorage;

[Exposed=(Window, Worker), SecureContext]
interface CrossOriginStorageManager {
  Promise<FileSystemFileHandle> requestFileHandle(
      CrossOriginStorageRequestFileHandleHash hash,
      CrossOriginStorageRequestFileHandleOptions options = {});
};

dictionary CrossOriginStorageRequestFileHandleHash {
  DOMString value;
  DOMString algorithm;
}

dictionary CrossOriginStorageRequestFileHandleOptions {
  DOMString description;
  optional boolean create = false;
}
```

### Appendix B: Blob hash with the Web Crypto API

```js
async function getBlobHash(blob) {
  const hashAlgorithmIdentifier = 'SHA-256';

  // Get the contents of the blob as binary data contained in an ArrayBuffer.
  const arrayBuffer = await blob.arrayBuffer();

  // Hash the arrayBuffer using SHA-256.
  const hashBuffer = await crypto.subtle.digest(
    hashAlgorithmIdentifier,
    arrayBuffer,
  );

  // Convert the ArrayBuffer to a hex string.
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  const hashHex = hashArray
    .map((byte) => byte.toString(16).padStart(2, '0'))
    .join('');

  return {
    algorithm: hashAlgorithmIdentifier,
    value: hashHex,
  };
}

// Example usage:
const fileBlob = await fetch('https://example.com/ai-model.bin').then(
  (response) => response.blob(),
);
getBlobHash(fileBlob).then((hash) => {
  console.log('Hash:', hash);
});
```

### Appendix C: FAQ

<details>
  <summary>
    <strong>Question:</strong> Does this API help with resuming downloads? What if downloading a large file fails before the file ends up in COS?
  </summary>
  <p>
    <strong>Answer:</strong> Managing downloads is out of scope of this proposal. COS can work with complete or with sharded files that the developer stores in COS as separate blobs and then assembles them after retrieval from COS. This way, downloads can be handled completely out-of-bounds, and developers can, for example, leverage the <a href="https://wicg.github.io/background-fetch/">Background Fetch API</a> or regular <code>fetch()</code> requests with <code>Range</code> headers to download large files.
  </p>
</details>

<details>
  <summary>
    <strong>Question:</strong> Why does this API not target popular JavaScript libraries like jQuery?
  </summary>
  <p>
    <strong>Answer:</strong> The short answer is version fragmentation. JavaScript libraries are way more fragmented than large AI models, SQLite databases, offline storage archives, and Wasm modules. The longer answer is that when the Chrome team did <a href="https://github.com/shivanigithub/http-cache-partitioning?tab=readme-ov-file#impact-on-metrics">research</a> in the context of <a href="https://github.com/shivanigithub/http-cache-partitioning">partitioning the HTTP cache</a>, they found that after partitioning the HTTP cache <em>"the overall cache miss rate increases by about 2 percentage points but changes to first contentful paint aren't statistically significant and the overall fraction of bytes loaded from the network only increase by around 1.5 percentage points"</em>. Furthermore, since the COS API <a href="#user-consent-and-permissions">requires permission</a> before accessing a file, it would, for the majority of web apps, not be practical to interrupt the user with a permission prompt for a few kilo- or megabytes of savings.
  </p>
</details>
