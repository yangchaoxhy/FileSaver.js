If you need to save really large files bigger than the blob's size limitation or don't have
enough RAM, then have a look at the more advanced [StreamSaver.js][7]
that can save data directly to the hard drive asynchronously with the power of the new streams API. That will have
support for progress, cancelation and knowing when it's done writing

FileSaver.js
============

FileSaver.js is the solution to saving files on the client-side, and is perfect for
web apps that generates files on the client, However if the file is coming from the
server we recommend you to first try to use [Content-Disposition][8] attachment response header as it has more cross-browser compatiblity.

Looking for `canvas.toBlob()` for saving canvases? Check out
[canvas-toBlob.js][2] for a cross-browser implementation.

Supported Browsers
------------------

| Browser        | Constructs as | Filenames    | Max Blob Size | Dependencies |
| -------------- | ------------- | ------------ | ------------- | ------------ |
| Firefox 20+    | Blob          | Yes          | 800 MiB       | None         |
| Firefox < 20   | data: URI     | No           | n/a           | [Blob.js](https://github.com/eligrey/Blob.js) |
| Chrome         | Blob          | Yes          | [2GB][3]      | None         |
| Chrome for Android | Blob      | Yes          | [RAM/5][3]    | None         |
| Edge           | Blob          | Yes          | ?             | None         |
| IE 10+         | Blob          | Yes          | 600 MiB       | None         |
| Opera 15+      | Blob          | Yes          | 500 MiB       | None         |
| Opera < 15     | data: URI     | No           | n/a           | [Blob.js](https://github.com/eligrey/Blob.js) |
| Safari 6.1+*   | Blob          | No           | ?             | None         |
| Safari < 6     | data: URI     | No           | n/a           | [Blob.js](https://github.com/eligrey/Blob.js) |
| Safari 10.1+   | Blob          | Yes          | n/a           | None         |

Feature detection is possible:

```js
try {
    var isFileSaverSupported = !!new Blob;
} catch (e) {}
```

### IE < 10

It is possible to save text files in IE < 10 without Flash-based polyfills.
See [ChenWenBrian and koffsyrup's `saveTextAs()`](https://github.com/koffsyrup/FileSaver.js#examples) for more details.

### Safari 6.1+

Blobs may be opened instead of saved sometimes—you may have to direct your Safari users to manually
press <kbd>⌘</kbd>+<kbd>S</kbd> to save the file after it is opened. Using the `application/octet-stream` MIME type to force downloads [can cause issues in Safari](https://github.com/eligrey/FileSaver.js/issues/12#issuecomment-47247096).

### iOS

saveAs must be run within a user interaction event such as onTouchDown or onClick; setTimeout will prevent saveAs from triggering. Due to restrictions in iOS saveAs opens in a new window instead of downloading, if you want this fixed please [tell Apple how this WebKit bug is affecting you](https://bugs.webkit.org/show_bug.cgi?id=167341).

Syntax
------
### Import `saveAs()` from file-saver
```js
import { saveAs } from 'file-saver';
```

```js
FileSaver saveAs(Blob/File/Url, optional DOMString filename, optional Object { autoBom })
```

Pass `{ autoBom: true }` if you want FileSaver.js to automatically provide Unicode text encoding hints (see: [byte order mark](https://en.wikipedia.org/wiki/Byte_order_mark)). Note that this is only done if your blob type has `charset=utf-8` set.

Examples
--------

### Saving text using `require()`
```js
var FileSaver = require('file-saver');
var blob = new Blob(["Hello, world!"], {type: "text/plain;charset=utf-8"});
FileSaver.saveAs(blob, "hello world.txt");
```

### Saving text

```js
var blob = new Blob(["Hello, world!"], {type: "text/plain;charset=utf-8"});
FileSaver.saveAs(blob, "hello world.txt");
```

### Saving URLs

```js
FileSaver.saveAs("https://httpbin.org/image", "image.jpg");
```
Using URLs within the same origin will just use `a[download]`.
Otherwise, it will first check if it supports cors header with a synchronous head request.
If it does, it will download the data and save using blob URLs.
If not, it will try to download it using `a[download]`.

The standard W3C File API [`Blob`][4] interface is not available in all browsers.
[Blob.js][5] is a cross-browser `Blob` implementation that solves this.

### Saving a canvas
```js
var canvas = document.getElementById("my-canvas");
canvas.toBlob(function(blob) {
    saveAs(blob, "pretty image.png");
});
```

Note: The standard HTML5 `canvas.toBlob()` method is not available in all browsers.
[canvas-toBlob.js][6] is a cross-browser `canvas.toBlob()` that polyfills this.

### Saving File

You can save a File constructor without specifying a filename. If the
file itself already contains a name, there is a hand full of ways to get a file
instance (from storage, file input, new constructor, clipboard event).
If you still want to change the name, then you can change it in the 2nd argument.

```js
// Note: Ie and Edge don't support the new File constructor,
// so it's better to construct blobs and use saveAs(blob, filename)
var file = new File(["Hello, world!"], "hello world.txt", {type: "text/plain;charset=utf-8"});
FileSaver.saveAs(file);
```

How It Works
------------

FileSaver.js selects the best available download mechanism at runtime based on browser capabilities. There are three mechanisms, applied in priority order:

### 1. `a[download]` — Anchor-element-to-UI (Primary mechanism)

The preferred approach on modern browsers. FileSaver.js creates a hidden `<a>` element, sets its `href` to an object URL (for Blobs) or the raw URL (for strings), and sets the `download` attribute to the desired filename. It then programmatically dispatches a `click` event to trigger the browser's native save/download UI.

```
[saveAs(blob, name)]
        |
        v
  Create <a download="name">
        |
        v
  a.href = URL.createObjectURL(blob)
        |
        v
  Dispatch click event → Browser download UI
        |
        v
  Revoke object URL after 40 seconds
```

Key details:
- Object URLs are revoked after 40 seconds to free memory.
- For cross-origin string URLs, a CORS check (`HEAD` request) is performed first. If the resource allows CORS access the file is fetched as a Blob and saved; otherwise the anchor is opened in a new tab.
- macOS WebView is excluded from this path because it misidentifies itself as a browser but does not support `a[download]`.

### 2. `msSaveOrOpenBlob` — Internet Explorer fallback

Used when `navigator.msSaveOrOpenBlob` is available (IE 10+). FileSaver.js delegates directly to the browser's built-in save API.

```
[saveAs(blob, name)]
        |
        v
  navigator.msSaveOrOpenBlob(blob, name) → IE Save dialog
```

For string URLs under IE, the same CORS check is applied: fetch as Blob if CORS is allowed, otherwise open in a new tab.

### 3. FileReader + popup — Legacy / iOS fallback

Used when neither `a[download]` nor `msSaveOrOpenBlob` is available (older Safari, Chrome on iOS, macOS WebView with `application/octet-stream` blobs).

A blank popup window is opened immediately (synchronously, within the user-interaction event) to avoid popup blockers. The Blob is then read asynchronously with a `FileReader`, and the resulting `data:` URI is navigated to inside that popup, causing the browser to offer the file for saving or display it.

```
[saveAs(blob, name)]
        |
        v
  open('', '_blank') → blank popup
        |
        v
  FileReader.readAsDataURL(blob)
        |
        v
  popup.location.href = data: URI → Browser handles download/display
```

### Byte Order Mark (BOM) handling

Before passing a Blob to any of the above mechanisms, FileSaver.js can optionally prepend a UTF-8 BOM (`0xEF 0xBB 0xBF`) when `{ autoBom: true }` is passed and the MIME type indicates a UTF-8 text or XML document. This ensures correct encoding detection by applications such as Microsoft Excel.

![Tracking image](https://in.getclicky.com/212712ns.gif)

  [1]: http://eligrey.com/demos/FileSaver.js/
  [2]: https://github.com/eligrey/canvas-toBlob.js
  [3]: https://bugs.chromium.org/p/chromium/issues/detail?id=375297#c107
  [4]: https://developer.mozilla.org/en-US/docs/DOM/Blob
  [5]: https://github.com/eligrey/Blob.js
  [6]: https://github.com/eligrey/canvas-toBlob.js
  [7]: https://github.com/jimmywarting/StreamSaver.js
  [8]: https://github.com/eligrey/FileSaver.js/wiki/Saving-a-remote-file#using-http-header

Installation
------------------

```bash
# Basic Node.JS installation
npm install file-saver --save
bower install file-saver
```

Additionally, TypeScript definitions can be installed via:

```bash
# Additional typescript definitions
npm install @types/file-saver --save-dev
```
