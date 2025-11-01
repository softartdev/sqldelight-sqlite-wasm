# Adding OPFS Support for SQLDelight on wasmJs

This guide explains how to enable Origin-Private FileSystem (OPFS) support for SQLDelight in a Kotlin Multiplatform project targeting `wasmJs`. OPFS allows your web application to have a persistent database that survives page reloads.

## Introduction to OPFS

The Origin-Private FileSystem is a storage space private to the origin of the web application. It is more powerful than other browser storage mechanisms like IndexedDB or localStorage for database workloads, offering better performance and reliability.

Using OPFS with SQLDelight allows for a persistent SQLite database directly in the browser.

## Prerequisites

OPFS has two main requirements:
1.  A secure context (the page must be served over HTTPS).
2.  Specific Cross-Origin-Embedder-Policy (COEP) and Cross-Origin-Opener-Policy (COOP) HTTP headers must be set.

This guide will walk you through setting up everything.

## Step 1: Download and Extract SQLite for Wasm

The official SQLite for WebAssembly build is not distributed on npm, so we need to download it manually as part of the Gradle build process.

In your `build.gradle.kts`, add the `de.undercouch.download` plugin:
```kotlin
plugins {
    id("de.undercouch.download") version "5.3.0"
}
```

Then, add the following tasks to download and unzip the SQLite wasm build. These tasks will download the zip, extract the necessary files, and place them in your project's resources.

```kotlin
import de.undercouch.gradle.tasks.download.Download

// ...

// See https://sqlite.org/download.html for the latest wasm build version
val sqliteVersion = 3400000 // Use the latest version from sqlite.org

val sqliteDownload = tasks.register("sqliteDownload", Download::class.java) {
  src("https://sqlite.org/2022/sqlite-wasm-$sqliteVersion.zip")
  dest(layout.buildDirectory.dir("tmp"))
  onlyIfModified(true)
}

val sqliteUnzip = tasks.register("sqliteUnzip", Copy::class.java) {
  dependsOn(sqliteDownload)
  from(zipTree(layout.buildDirectory.dir("tmp/sqlite-wasm-$sqliteVersion.zip"))) {
    include("sqlite-wasm-$sqliteVersion/jswasm/**")
    exclude("**/*worker1*") // We use our own worker

    eachFile {
      relativePath = RelativePath(true, *relativePath.segments.drop(2).toTypedArray())
    }
  }
  into(layout.buildDirectory.dir("sqlite"))
  includeEmptyDirs = false
}

// Hook the unzip task into the jsProcessResources task
tasks.named("jsProcessResources").configure {
  dependsOn(sqliteUnzip)
}

kotlin {
  // ...
  sourceSets {
    val jsMain by getting {
      // ...
      resources.srcDir(layout.buildDirectory.dir("sqlite"))
    }
  }
}
```

This configuration does the following:
1.  Downloads the sqlite-wasm zip file.
2.  Extracts only the `jswasm` directory.
3.  Places the contents of `jswasm` into `build/sqlite`, which is then added as a `jsMain` resource directory.

## Step 2: Configure the SQLDelight Web Worker for OPFS

SQLDelight's `WebWorkerDriver` uses a worker script to run database operations off the main thread. We need to create a custom worker script to configure SQLite for OPFS.

Create a file named `sqlite.worker.js` in your `src/jsMain/resources` directory with the following content:

```javascript
importScripts("sqlite3.js");

let db = null;

async function createDatabase() {
  const sqlite3 = await sqlite3InitModule();

  // This is the key part for OPFS support
  // It instructs SQLite to use the OPFS VFS.
  db = new sqlite3.oo1.DB("file:database.db?vfs=opfs", "c");
}

function handleMessage() {
  const data = this.data;

  switch (data && data.action) {
    case "exec":
      if (!data["sql"]) {
        throw new Error("exec: Missing query string");
      }

      return postMessage({
        id: data.id,
        results: { values: db.exec({ sql: data.sql, bind: data.params, returnValue: "resultRows" }) },
      })
    case "begin_transaction":
      return postMessage({
        id: data.id,
        results: db.exec("BEGIN TRANSACTION;"),
      })
    case "end_transaction":
      return postMessage({
        id: data.id,
        results: db.exec("END TRANSACTION;"),
      })
    case "rollback_transaction":
      return postMessage({
        id: data.id,
        results: db.exec("ROLLBACK TRANSACTION;"),
      })
    default:
      throw new Error(`Unsupported action: ${data && data.action}`);
  }
}

function handleError(err) {
  return postMessage({
    id: this.data.id,
    error: err,
  });
}

if (typeof importScripts === "function") {
  db = null;
  const sqlModuleReady = createDatabase();
  self.onmessage = (event) => {
    return sqlModuleReady.then(handleMessage.bind(event))
    .catch(handleError.bind(event));
  }
}
```
The important line is `db = new sqlite3.oo1.DB("file:database.db?vfs=opfs", "c");`. This creates a database named `database.db` using the OPFS virtual file system.

## Step 3: Initialize the Web Worker Driver

Now, in your Kotlin code, initialize the `WebWorkerDriver` with the custom worker script.

```kotlin
import app.cash.sqldelight.db.SqlDriver
import app.cash.sqldelight.driver.worker.WebWorkerDriver
import org.w3c.dom.Worker

// ...

val worker = Worker(js("""new URL("sqlite.worker.js", import.meta.url)""").unsafeCast<String>())
val driver: SqlDriver = WebWorkerDriver(worker)

// Now you can create your database and use the driver
// Database.Schema.awaitCreate(driver)
// val database = Database(driver)
```

## Step 4: Configure HTTP Headers for OPFS

As mentioned, OPFS requires specific HTTP headers.

### For Local Development

For the Kotlin/JS webpack development server, you can add the headers by creating a file `webpack.config.d/opfs.js` with the following content:

```javascript
config.devServer = {
  ...config.devServer,
  headers: {
    "Cross-Origin-Embedder-Policy": "require-corp",
    "Cross-Origin-Opener-Policy": "same-origin",
  }
}
```

### For Deployment (GitHub Pages and other static hosts)

Many static hosting platforms, including GitHub Pages, do not allow you to set custom HTTP headers. To work around this, you can use the `coi-serviceworker` library. It's a service worker that intercepts requests and adds the necessary headers.

1.  **Download the service worker**:
    Add it to your `src/jsMain/resources` directory.
    ```bash
    curl -o src/jsMain/resources/coi-serviceworker.js https://raw.githubusercontent.com/gzuidhof/coi-serviceworker/master/coi-serviceworker.js
    ```

2.  **Include it in your `index.html`**:
    Add this script tag to the `<head>` of your `src/jsMain/resources/index.html`. It should be one of the first scripts to run.
    ```html
    <head>
      <!-- ... other head elements ... -->
      <script src="coi-serviceworker.js"></script>
    </head>
    ```

The `coi-serviceworker` will automatically activate and reload the page once, enabling the required headers and allowing OPFS to work.

## Conclusion

With these steps, you have configured your Kotlin/WasmJs application to use SQLDelight with a persistent SQLite database using OPFS. Your application's data will now persist between browser sessions.
