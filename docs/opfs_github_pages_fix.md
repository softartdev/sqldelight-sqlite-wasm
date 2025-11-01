# OPFS SQLite Persistence on GitHub Pages - Problem and Solution

## Problem Description

When deploying a SQLDelight + SQLite WASM web application to GitHub Pages, the database failed to work with OPFS (Origin Private File System) persistence. The application worked locally but failed on GitHub Pages with the following errors:

```
sqlite3.js:9978 Ignoring inability to install OPFS sqlite3_vfs: This environment does not have OPFS support.
CoroutineExceptionHandlerImpl.kt:7 J: {"message":"sqlite result code 1: no such vfs: opfs","name":"Error"}
```

### Root Cause

OPFS requires `SharedArrayBuffer` to function, which in turn requires specific HTTP headers to be set:
- `Cross-Origin-Opener-Policy: same-origin`
- `Cross-Origin-Embedder-Policy: require-corp`

**GitHub Pages does not allow setting custom HTTP headers**, making OPFS unavailable by default. This prevents persistent database storage in SQLite WASM applications deployed on GitHub Pages.

## Solution

### 1. Add COI Service Worker (`coi-serviceworker`)

The primary solution uses the [`coi-serviceworker`](https://github.com/gzuidhof/coi-serviceworker) library, which enables COOP/COEP headers via a service worker that intercepts fetch requests and adds the required headers.

**Implementation:**

1. **Download and add the service worker script:**
   ```bash
   curl -L -o src/jsMain/resources/coi-serviceworker.js \
     https://raw.githubusercontent.com/gzuidhof/coi-serviceworker/master/coi-serviceworker.js
   ```

2. **Register in `index.html`:**
   ```html
   <head>
       <!-- COI Service Worker to enable COOP/COEP headers for SharedArrayBuffer/OPFS -->
       <script src="coi-serviceworker.js"></script>
   </head>
   ```

The service worker automatically:
- Registers itself on first page load
- Reloads the page to activate
- Intercepts all fetch requests and adds COOP/COEP headers
- Enables `SharedArrayBuffer` and OPFS support

## Files Changed

### Essential Changes

1. **`src/jsMain/resources/coi-serviceworker.js`** (NEW)
   - Service worker library that enables COOP/COEP headers

2. **`src/jsMain/resources/index.html`**
   - Added `<script src="coi-serviceworker.js"></script>` in `<head>`

3. **`.github/workflows/site.yaml`**
   - Updated artifact path to `build/dist/js/productionExecutable`

## Result

✅ OPFS SQLite persistence now works on GitHub Pages  
✅ Database survives page reloads  
✅ All SQLite WASM files are correctly deployed  
✅ Service worker provides required headers for SharedArrayBuffer  
✅ No fallback to in-memory database needed  

## Testing

To verify the solution works:

1. Deploy to GitHub Pages
2. Save a note in the application
3. Reload the page
4. Verify the note persists (confirms OPFS is working)

## Alternative Solutions

If service workers are not viable for your use case, consider:

1. **Netlify**: Supports custom headers via `_headers` file
2. **Vercel**: Supports custom headers via `vercel.json`
3. **Firebase Hosting**: Supports custom headers configuration

## References

- [coi-serviceworker GitHub Repository](https://github.com/gzuidhof/coi-serviceworker)
- [SQLite WASM OPFS Documentation](https://sqlite.org/wasm/doc/trunk/persistence.md#opfs)
- [MDN: Cross-Origin Isolation](https://developer.mozilla.org/en-US/docs/Web/API/crossOriginIsolated)
- [MDN: SharedArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)

## Notes for AI Agents

When encountering similar issues with OPFS/SharedArrayBuffer on static hosting platforms:

1. **Check for header support**: If the platform doesn't support custom headers, use `coi-serviceworker`
2. **Verify file deployment**: Ensure all required files are included in the build output
3. **Service worker lifecycle**: The service worker may require a page reload to activate (handled automatically by `coi-serviceworker`)

