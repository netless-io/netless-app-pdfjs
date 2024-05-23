# @netless/app-pdfjs

A [Netless App](https://github.com/netless-io/netless-app) that renders PDF files with [PDF.js](https://github.com/mozilla/pdf.js).

## Install

<pre>npm add <strong>@netless/app-pdfjs</strong></pre>

## Usage

> [!IMPORTANT]
> This app only implements viewing PDF files.\
> It does not support `dispatchDocsEvent()` nor <q>export PDF</q>.

1. Get a static URL pointing to the PDF file.

   This package only synces the URL for each client to download the PDF.
   You have to obtain a static URL to the file first to continue.
   For example, you can use an <abbr title="Object Storage Service">OSS</abbr> to achieve this.

2. Register this app **before** joinning room.

   ```js
   import { register } from "@netless/fastboard"
   import { install } from "@netless/app-pdfjs"

   install(register) // the app is named 'PDFjs'
   ```

3. Add this app **after** joinning room.

   ```js
   fastboard.manager.addApp({
     kind: 'PDFjs',
     options: {
       title: 'a.pdf',
       scenePath: '/pdf/paper' // ! This is required.
     },
     attributes: {
       src: 'https://cdn.jsdelivr.net/gh/mfogel/polygon-clipping/paper.pdf'
     }
   })
   ```

## Troubleshooting

### Failed to fetch pdf.min.mjs

This package downloads the latest PDF.js release from jsDelivr:\
<samp>https://cdn.jsdelivr.net/npm/pdfjs-dist@latest/build/</samp>

To alter this URL or choose a different version, set the app option:

```js
install(register, {
  appOptions: {
    pdfjsLib: 'https://cdn.jsdelivr.net/npm/pdfjs-dist@latest/build/pdf.min.mjs',
    workerSrc: 'https://cdn.jsdelivr.net/npm/pdfjs-dist@latest/build/pdf.worker.min.mjs'
  }
})
```

If the URL is blocked by your [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP),
you have to add them to your `script-src` policy:

```
Content-Security-Policy: script-src https://cdn.jsdelivr.net/
```

### TypeError: Promise.withResolvers is not a function

PDF.js uses [`Promise.withResolvers()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/withResolvers),
which is a brand new feature in JavaScript.
In case your target environment does not support this,
You need to add the following code to the first line of 
`pdf.min.mjs` and `pdf.worker.min.mjs` to resolve this issue.

```javascript
if (typeof Promise.withResolvers === 'undefined') {
  Promise.withResolvers = function () {
    let resolve, reject
    const promise = new Promise((res, rej) => {
      resolve = res
      reject = rej
    })
    return { promise, resolve, reject }
  }
}
```
Alternatively, you can run `node ./scripts/generatorPolyfillForPdfjs.mjs`.
This command will generate the latest version of the PDF.js files with polyfills applied.
Upload these two files to a file hosting platform for use.


### Uncaught SyntaxError / Uncaught ReferenceError

This package only targets modern browsers that support [native ES modules](https://caniuse.com/es6-module).

### PDF Rendering Issue

As the package name implies, it uses PDF.js to render the file.
It is possible that PDF.js has a bug when rendering some files.
Please raise an issue [there](https://github.com/mozilla/pdf.js) to ask for help.

## License

MIT @ [netless](https://github.com/netless-io)
