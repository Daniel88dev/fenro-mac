# Local attachment and design-artifact storage under App Sandbox

Research for issue #4. Assumptions: App Sandbox is on, distribution is a direct
notarized download (not the Mac App Store), no backend exists yet. Sources are
Apple documentation unless marked otherwise. Anything Apple's docs do not state
outright is marked **UNVERIFIED**.

## Summary

- Yes, a sandboxed macOS app can store attachments locally with no backend. The
  simplest, most robust approach is to **copy the user's file into the app's own
  container** (`Application Support` inside `~/Library/Containers/<bundle-id>/Data`),
  which the app can read and write without any extra entitlement.
- **Security-scoped bookmarks** let the app keep pointing at a file *in place*
  (outside the container) across relaunches, but they cost an entitlement, a
  start/stop access dance around every use, stale-bookmark handling, and they
  break when the user deletes the original. Good for "link to a big file I
  don't want duplicated"; poor as the primary attachment mechanism.
- `WKWebView.loadFileURL(_:allowingReadAccessTo:)` renders a local HTML/JS/CSS
  bundle. Pass the bundle's *directory* as the read-access URL so sibling JS and
  CSS files load. Files in the container need no extra entitlement. Scripts run
  normally; only remote resources would need the network-client entitlement.
- `NSWorkspace.shared.open(url)` opens a file URL in the default app for that
  type (a browser for `.html`). It works for files inside the container in
  practice; Apple's docs do not spell out the sandbox-extension mechanics, so
  that detail is UNVERIFIED.
- Recommended layout: content-addressed blobs (`SHA-256` hex name) under
  `Application Support/<bundle-id>/blobs/`, with one metadata row per attachment
  in the main store. Dedup is free, sync is a later add-on, cost is small.

## 1. Storing user files: copy into the container vs. bookmark in place

### What the sandbox gives you for free

- "The operating system creates a container directory when launching your
  sandboxed app, to which the app has unrestricted read and write access. The
  sandboxed app does not have unrestricted access to the user's home folder."
  ([Protecting user data with App Sandbox](https://developer.apple.com/documentation/security/protecting-user-data-with-app-sandbox))
- "Use the `FileManager` method `url(for:in:appropriateFor:create:)` to find
  common directories ... For a sandboxed app, this method returns a location
  within the app's container rather than the user's home directory."
  (same page)
- The container lives under `~/Library/Containers` and is tied to the app's
  code signature. ([Accessing files from the macOS App Sandbox](https://developer.apple.com/documentation/security/accessing-files-from-the-macos-app-sandbox))
- `FileManager.SearchPathDirectory.applicationSupportDirectory` maps to
  `Library/Application Support`. ([applicationSupportDirectory](https://developer.apple.com/documentation/foundation/filemanager/searchpathdirectory/applicationsupportdirectory))
  Apple's convention: put your files in a subdirectory named after the bundle
  identifier, and "Your app is responsible for creating this directory as
  needed." ([File System Programming Guide, macOS Library directories](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/MacOSXDirectories/MacOSXDirectories.html))

Swift idiom (no entitlement needed):

```swift
let base = try FileManager.default.url(
    for: .applicationSupportDirectory, in: .userDomainMask,
    appropriateFor: nil, create: true)
let blobs = base.appendingPathComponent("blobs", isDirectory: true)
try FileManager.default.createDirectory(at: blobs, withIntermediateDirectories: true)
```

### Getting the user's file in the first place

Whichever approach you pick, the user must hand the app a file through an
Open panel (SwiftUI `fileImporter`, AppKit `NSOpenPanel`) or drag and drop.
"The system automatically extends the app's sandbox to include selected URLs";
picking a folder extends access to its contents. This requires the
`com.apple.security.files.user-selected.read-write` entitlement (or the
`read-only` variant): "A Boolean value that indicates whether the app may have
read-write access to files the user has selected using an Open or Save dialog."
Enable it in Xcode via Signing & Capabilities, App Sandbox, User Selected File.
([entitlement page](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.files.user-selected.read-write),
[Accessing files](https://developer.apple.com/documentation/security/accessing-files-from-the-macos-app-sandbox))

Access granted by the panel lasts only for the current process. If the app is
relaunched, it is gone unless you (a) copied the file into the container, or
(b) saved a security-scoped bookmark.

### Option A: copy into the container

- Read the picked URL while the panel grant is live, hash the bytes, write them
  to `Application Support/<bundle-id>/blobs/<sha256>`.
- After the copy, the app owns the bytes forever. The original moving, being
  renamed, deleted, or living on an unmounted external drive does not matter.
- No further entitlement, no start/stop access, no stale handling.
- Cost: disk space is duplicated. For design artifacts (HTML bundles, images,
  PDFs) this is normally small; for multi-GB video it is not.

### Option B: security-scoped bookmark to the file in place

A bookmark is an opaque `Data` blob that Foundation can turn back into a URL
later "even if the user moves or renames it (if the volume format on which the
file resides supports doing so)". ([bookmarkData(options:includingResourceValuesForKeys:relativeTo:)](https://developer.apple.com/documentation/foundation/nsurl/bookmarkdata(options:includingresourcevaluesforkeys:relativeto:)))

**Entitlement.** Persistent access needs
`com.apple.security.files.bookmarks.app-scope` (true) in the `.entitlements`
file (or `document-scope` for bookmarks stored inside a document file). "If you
want to provide your sandboxed app with persistent access to file system
resources, you must enable security-scoped bookmark and URL access."
([Entitlement Key Reference, Enabling App Sandbox](https://developer.apple.com/library/archive/documentation/Miscellaneous/Reference/EntitlementKeyReference/Chapters/EnablingAppSandbox.html))
The bookmark entitlements are not in the modern `bundleresources/entitlements`
index (that URL 404s); the archived reference is the authoritative page.

**Creating.** Include `.withSecurityScope`; add
`.securityScopeAllowOnlyReadAccess` if the app only ever needs to read. Pass
`relativeTo: nil` for an app-scoped bookmark. `.withSecurityScope` "can't be
used in conjunction with either `minimalBookmark` or `suitableForBookmarkFile`".
([withSecurityScope](https://developer.apple.com/documentation/foundation/nsurl/bookmarkcreationoptions/withsecurityscope),
[bookmarkData](https://developer.apple.com/documentation/foundation/nsurl/bookmarkdata(options:includingresourcevaluesforkeys:relativeto:)))

```swift
// While the open-panel grant is live:
let bookmark = try url.bookmarkData(
    options: [.withSecurityScope],
    includingResourceValuesForKeys: nil, relativeTo: nil)
// persist `bookmark` (Data) in the main store
```

**Resolving and the stale flag.** Resolve with `.withSecurityScope` and check
`bookmarkDataIsStale`. Apple's sample: if stale, immediately re-create the
bookmark from the resolved URL and overwrite the stored copy.
([Accessing files](https://developer.apple.com/documentation/security/accessing-files-from-the-macos-app-sandbox),
[init(resolvingBookmarkData:...)](https://developer.apple.com/documentation/foundation/nsurl/init(resolvingbookmarkdata:options:relativeto:bookmarkdataisstale:)))

```swift
var isStale = false
let url = try URL(resolvingBookmarkData: bookmark,
                  options: [.withSecurityScope],
                  relativeTo: nil, bookmarkDataIsStale: &isStale)
if isStale {
    let fresh = try url.bookmarkData(options: [.withSecurityScope],
                                     includingResourceValuesForKeys: nil, relativeTo: nil)
    // store `fresh`
}
guard url.startAccessingSecurityScopedResource() else { /* access denied */ return }
defer { url.stopAccessingSecurityScopedResource() }
// read or write the file here
```

**The start/stop rule.** "When you obtain a security-scoped URL, such as by
resolving a security-scoped bookmark, you can't immediately use the resource it
points to." You must call `startAccessingSecurityScopedResource()`, and "You
must balance each call ... with a call to
`stopAccessingSecurityScopedResource()`." The warning matters: "If you fail to
relinquish your access ... your app leaks kernel resources. If sufficient
kernel resources leak, your app loses its ability to add file-system locations
to its sandbox, such as with Powerbox or security-scoped bookmarks, until
relaunched." ([startAccessingSecurityScopedResource()](https://developer.apple.com/documentation/foundation/nsurl/startaccessingsecurityscopedresource()))
URLs that come straight from an Open panel are already "started" for you; you
only need `stop` when done. ([Accessing files](https://developer.apple.com/documentation/security/accessing-files-from-the-macos-app-sandbox))

**Who can use the bookmark.** App-scoped bookmarks are bound to the code-signing
identity: "A bookmark created with security scope fails to resolve if the
caller does not have the same code signing identity as the caller that created
the bookmark." ([bookmarkData](https://developer.apple.com/documentation/foundation/nsurl/bookmarkdata(options:includingresourcevaluesforkeys:relativeto:)))
Consequence: bookmarks are machine- and app-identity-specific. They cannot be
synced to another Mac or backend and mean anything there.

**When the original moves or is deleted.**

- Moved or renamed on the same volume: resolves (bookmarks track the file by
  identity where the volume supports it), typically with `isStale == true`, so
  re-save it. ([bookmarkData](https://developer.apple.com/documentation/foundation/nsurl/bookmarkdata(options:includingresourcevaluesforkeys:relativeto:)))
- Deleted, or on a volume that is not mounted: resolution throws. Apple's pages
  fetched here do not name the specific error code; **UNVERIFIED** that it is
  `NSFileNoSuchFileError`. Either way, the app must show a "missing file" state.
- Moved to a different volume: **UNVERIFIED** in Apple docs; expect the
  bookmark to fail, since the tracking is per-volume.

### Backup behaviour of the container

- Time Machine backs up user files generally: "you can use Time Machine to
  automatically back up your files, including apps, music, photos, email, and
  documents." ([Apple Support 104984](https://support.apple.com/en-us/104984))
  The container is under `~/Library`, so it is included by default. Apple's
  published docs fetched here do not list the built-in exclusions
  (`~/Library/Caches` etc.); that they are excluded while `Application Support`
  is included is **UNVERIFIED** from primary sources but is long-standing
  behaviour. Attachments belong in `Application Support`, not `Caches`, so they
  are backed up either way.
- iCloud: the "Desktop & Documents Folders" feature syncs only those two folders
  and says nothing about `~/Library` or containers. ([Apple Support 109344](https://support.apple.com/en-us/109344))
  There is no iCloud backup of a Mac app's container (iCloud Backup is an iOS
  feature). If cloud sync of attachments is wanted later it has to be built
  (CloudKit or the project's own backend).
- To keep a regenerable file out of backups, set `isExcludedFromBackupKey` on
  the URL; "Set this property each time you save a file because some common
  file operations cause this property to reset to `false`."
  ([isExcludedFromBackupKey](https://developer.apple.com/documentation/foundation/urlresourcekey/isexcludedfrombackupkey))
  Use this for derived thumbnails and caches, not for the attachments
  themselves.

### Comparison

| | Copy into container | Security-scoped bookmark |
|---|---|---|
| Extra entitlement | none | `files.bookmarks.app-scope` |
| Survives relaunch | yes | yes, if resolved and re-saved when stale |
| Survives original move/rename | yes | usually (same volume) |
| Survives original delete | yes | no |
| Disk cost | duplicated bytes | none |
| Code complexity | low | medium (start/stop, stale, missing states) |
| Syncable to another machine | yes (bytes are yours) | no (identity-bound) |
| Renderable in WKWebView | direct | must wrap every load in start/stop |

## 2. Rendering an HTML, JS and CSS bundle in WKWebView

### `loadFileURL(_:allowingReadAccessTo:)`

Signature: `func loadFileURL(_ URL: URL, allowingReadAccessTo readAccessURL: URL) -> WKNavigation?`
(macOS 10.11+). `readAccessURL` is "The URL of a file or directory containing
web content that you grant the system permission to read." "To prevent WebKit
from reading any other content, specify the same value as the URL parameter. To
read additional files related to the content file, specify a directory."
([loadFileURL](https://developer.apple.com/documentation/webkit/wkwebview/loadfileurl(_:allowingreadaccessto:)))

So for a bundle laid out as `index.html`, `app.js`, `style.css` in one folder,
pass the folder as `readAccessURL`. The page's relative `<script src="app.js">`
and `<link href="style.css">` then resolve and are readable. Passing only the
HTML file's URL makes the JS and CSS fail to load. WebKit runs page content in
a separate process, and this parameter is what lets that process read the
files; it is the only sandbox-related knob you need.

```swift
let dir  = blobs.appendingPathComponent(artifactID, isDirectory: true)
let page = dir.appendingPathComponent("index.html")
webView.loadFileURL(page, allowingReadAccessTo: dir)
```

### Entitlements

- Files inside the app container need **no extra entitlement**; the container is
  already inside the sandbox ([Protecting user data](https://developer.apple.com/documentation/security/protecting-user-data-with-app-sandbox)).
- If the bundle is on a user-selected path outside the container, the app must
  hold access to it (panel grant or resolved bookmark with
  `startAccessingSecurityScopedResource()` held for the life of the load).
- If the HTML references remote resources (CDN scripts, fonts, analytics), the
  app needs `com.apple.security.network.client`: "A Boolean value indicating
  whether your app may open outgoing network connections."
  ([network.client](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.network.client))
  A purely local bundle does not.

### Local JavaScript

- Inline and same-folder scripts execute normally under `loadFileURL`. Apple's
  docs impose no sandbox-specific restriction on local JavaScript.
- **UNVERIFIED** (WebKit behaviour, not documented by Apple's API pages): pages
  loaded from `file://` have an opaque origin, so `fetch()`/`XMLHttpRequest` to
  other `file://` URLs, `localStorage` persistence and some cross-origin
  features behave differently from `http://`. If an artifact relies on
  `fetch("data.json")`, expect it to fail from `file://`.
- The documented escape hatch is a custom scheme: register a
  `WKURLSchemeHandler` on `WKWebViewConfiguration` with
  `setURLSchemeHandler(_:forURLScheme:)` (macOS 10.13+) and serve bundle files
  yourself under e.g. `fenro-artifact://<id>/index.html`. The scheme "must start
  with an ASCII letter", may not be one WebKit already handles (`https`,
  `file`), and Apple's tip is to "include the name of your app or company in any
  custom scheme names." ([setURLSchemeHandler](https://developer.apple.com/documentation/webkit/wkwebviewconfiguration/seturlschemehandler(_:forurlscheme:)),
  [WKURLSchemeHandler](https://developer.apple.com/documentation/webkit/wkurlschemehandler))
  This gives every artifact a proper origin, so `fetch` works, and it also lets
  the app serve content-addressed blobs from any layout without exposing real
  paths. It is a small amount of Swift (implement `webView(_:start:)` to read a
  file and reply with the bytes and a MIME type).
- `loadHTMLString(_:baseURL:)` is for HTML you generate in code; the `baseURL`
  only affects how relative URLs resolve, and it does not grant file read
  access. Prefer `loadFileURL` or a scheme handler for bundles on disk.
  ([loadHTMLString](https://developer.apple.com/documentation/webkit/wkwebview/loadhtmlstring(_:baseurl:)))

### Opening the same bundle in the default browser

- `NSWorkspace.shared.open(_ url: URL) -> Bool` "Opens the location at the
  specified URL." The async form `open(_:configuration:completionHandler:)`
  hands back the `NSRunningApplication` that opened it. Neither page states in
  words that a file URL goes to the default app for its type, but that is what
  the whole `NSWorkspace` API family (including
  `setDefaultApplication(at:toOpen:...)`) is built around.
  ([open(_:)](https://developer.apple.com/documentation/appkit/nsworkspace/open(_:)),
  [open(_:configuration:completionHandler:)](https://developer.apple.com/documentation/appkit/nsworkspace/open(_:configuration:completionhandler:)),
  [NSWorkspace](https://developer.apple.com/documentation/appkit/nsworkspace))
- For a file inside the container: **UNVERIFIED** from Apple docs, but
  Launch Services passes the receiving app access to the specific file when you
  open it this way, and Safari/Chrome opening an `index.html` from another app's
  container works in practice. The browser will then request `app.js` and
  `style.css` by relative path; since the browser is not sandboxed the same way
  (or receives access to the item it was handed), this usually works for
  Safari. Treat "works in every browser" as something to test, not assume.
- Alternative that avoids the question: `activateFileViewerSelecting(_:)`
  reveals the file in Finder and lets the user double-click it.
  ([NSWorkspace](https://developer.apple.com/documentation/appkit/nsworkspace))
- If the bundle ever needs `fetch()` to work in the external browser too,
  export it (copy to a user-chosen location via `NSSavePanel`) rather than
  opening the container path.

## 3. Cost of a sync-friendly storage layout

Reasoning, not fetched.

**Layout**

```
Application Support/<bundle-id>/
  blobs/<first-2-hex>/<sha256-hex>      # immutable bytes, no extension
  main store (SQLite / SwiftData / whatever the app uses)
    attachments: id, sha256, byte_count, mime_type, original_filename,
                 created_at, owner_entity_id, kind (file | html-bundle)
    bundle_members (for HTML artifacts): bundle_id, relative_path, sha256
```

**Why content-addressed**

- Dedup is free: importing the same PDF twice writes one blob and two rows.
- Integrity check is free: rehash the file and compare.
- Immutability makes sync trivial: a blob never changes, so a backend only
  needs "do you have `<hash>`? no? here it is." No conflict resolution on bytes,
  only on metadata rows, which the main store's sync already has to solve.
- Deleting is a garbage-collect: remove the row, then delete any blob no row
  references.

**Cost**

- Hashing: `SHA256` from CryptoKit, streamed in chunks, roughly the cost of
  reading the file once. Negligible for design artifacts.
- Code: one `BlobStore` type (put(Data or URL) -> hash, url(for: hash),
  gc(referenced: Set<hash>)) is about 100 to 150 lines including tests.
- HTML bundles: either store each member file as its own blob and materialise a
  directory on demand for `loadFileURL`, or store the bundle as one zip blob and
  unpack into `Caches/` (excluded from backup) when rendering. With a custom
  `WKURLSchemeHandler` you skip materialising entirely and serve members
  straight from the blob store by relative path. Recommended: per-member blobs +
  scheme handler.
- Extensionless blob names mean `NSWorkspace.open` on a raw blob will not pick
  a good app. Keep `original_filename`, and when opening externally, hard-link
  or copy the blob to `Caches/exports/<id>/<original_filename>` first.

**What a future backend sync needs**

- Content-addressed blob upload/download keyed by SHA-256 (any object store).
- Metadata rows already carry the hash, so the sync layer is "push rows, then
  push missing blobs". No schema change needed later.
- Security-scoped bookmarks would *not* sync (identity-bound, machine-bound), which
  is the main reason not to make them the primary mechanism.

## Implications for a later attachments effort

**Recommended approach**

1. Primary mechanism: **copy into the container, content-addressed**. Zero extra
   entitlements beyond `user-selected.read-write` for the Open panel, no
   start/stop access, no stale state, survives anything happening to the
   original, backed up by Time Machine, sync-ready.
2. Render HTML artifacts with `WKWebView`. Start with
   `loadFileURL(index, allowingReadAccessTo: bundleDirectory)` for speed; move to
   a `fenro-artifact://` `WKURLSchemeHandler` when artifacts need `fetch()` or
   when serving from per-member blobs becomes more convenient than
   materialising folders.
3. "Open in browser": `NSWorkspace.shared.open(url)` on a materialised copy under
   `Caches/exports/<id>/` with real filenames. Test in Safari and Chrome before
   promising it.
4. Offer security-scoped bookmarks later as an opt-in "link instead of copy" for
   large files only. If that ships: add `files.bookmarks.app-scope`, wrap every
   access in start/stop with `defer`, re-save on stale, and design a
   "missing file" UI state.
5. Keep thumbnails and unpacked bundles in `Caches/` with
   `isExcludedFromBackupKey` set on write.

**Things to verify when building (not verifiable from docs alone)**

- Behaviour of `fetch()`/`XMLHttpRequest` from `file://` pages in WKWebView.
- Exact error thrown when a bookmarked file has been deleted.
- Which browsers open `index.html` from inside the container and load siblings.
- Time Machine's built-in exclusion list for `~/Library/Caches` on current macOS.

## Sources

- https://developer.apple.com/documentation/security/app-sandbox
- https://developer.apple.com/documentation/security/protecting-user-data-with-app-sandbox
- https://developer.apple.com/documentation/security/accessing-files-from-the-macos-app-sandbox
- https://developer.apple.com/documentation/foundation/nsurl/bookmarkdata(options:includingresourcevaluesforkeys:relativeto:)
- https://developer.apple.com/documentation/foundation/nsurl/bookmarkcreationoptions/withsecurityscope
- https://developer.apple.com/documentation/foundation/nsurl/init(resolvingbookmarkdata:options:relativeto:bookmarkdataisstale:)
- https://developer.apple.com/documentation/foundation/nsurl/startaccessingsecurityscopedresource()
- https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.files.user-selected.read-write
- https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.network.client
- https://developer.apple.com/library/archive/documentation/Miscellaneous/Reference/EntitlementKeyReference/Chapters/EnablingAppSandbox.html
- https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/MacOSXDirectories/MacOSXDirectories.html
- https://developer.apple.com/documentation/foundation/filemanager/searchpathdirectory/applicationsupportdirectory
- https://developer.apple.com/documentation/foundation/urlresourcekey/isexcludedfrombackupkey
- https://developer.apple.com/documentation/webkit/wkwebview/loadfileurl(_:allowingreadaccessto:)
- https://developer.apple.com/documentation/webkit/wkwebview/loadhtmlstring(_:baseurl:)
- https://developer.apple.com/documentation/webkit/wkurlschemehandler
- https://developer.apple.com/documentation/webkit/wkwebviewconfiguration/seturlschemehandler(_:forurlscheme:)
- https://developer.apple.com/documentation/appkit/nsworkspace
- https://developer.apple.com/documentation/appkit/nsworkspace/open(_:)
- https://developer.apple.com/documentation/appkit/nsworkspace/open(_:configuration:completionhandler:)
- https://support.apple.com/en-us/104984
- https://support.apple.com/en-us/109344
