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

2. Convert this PDF file using the [Agora File Conversion](https://docs.agora.io/en/interactive-whiteboard/reference/whiteboard-api/file-conversion?platform=android#start-file-conversion) service. Remember to set `outputFormat: "qpdf"`.

   You will get a `taskId` and `prefix` URL in the response of [Query Task Status API](https://docs.agora.io/en/interactive-whiteboard/reference/whiteboard-api/file-conversion?platform=android#query-the-progress-of-a-file-conversion-task).

3. Register this app **before** joinning room.

   ```js
   import { register } from "@netless/fastboard"
   import { install } from "@netless/app-pdfjs"

   install(register, {
     // private bucket
     urlInterrupter: async (url: string, prefix: string, taskId: string) => {
       // There will be different implementations depending on different cloud storage services. Generally, signatures are added to the query parameters.
       // method 1: Add a signature to the query parameters. 
       const { ak, expire } = await getSTSToken() // Customer service side implementation.
       return `${url}?expire=${expire}&ak=${ak}`
       
       // method 2: Add a signature to the path.
       const url = await getTokenByParams(prefix, taskId) // Customer service side implementation.
       return url;
     }
   }) // the app is named 'PDFjs'
   ```

4. Add this app **after** joinning room.

   ```js
   fastboard.manager.addApp({
     kind: 'PDFjs',
     options: {
       title: 'a.pdf', // ! Required for window title.
       scenePath: '/pdf/paper' // ! Required for displaying whiteboard on it.
     },
     attributes: {
       prefix, // ! Required.
       taskId, // ! Required.
     }
   })
   ```

   - Make sure `scenePath` starts with `/` and not ends with `/`.
   - The `prefix` will be like `"https://white-cover.oss-cn-hangzhou.aliyuncs.com/flat/"`.
   - The `taskId` will be like `"b444a180c2f44a409a4d081e8f1a6d5f"`.

   You can get the `prefix` and `taskId` from the conversion response JSON.

### notice
>1. When not using urlinterrupter, pdfjs app generates the url by taking 2 parameters prefix and taskid and concatenate them in a fixed format. If you need different url format, you can use urlinterrupter. When not using urlinterrupter, pdfjs app generates the url by taking 2 parameters prefix and taskid and concatenate them in a fixed format. Please see the example as below:
>>  ```js
>>  Prefix: https://testbucket.s3.ap-southeast-1.amazonaws.com/whiteboard
>>  Taskid: e349fe51afb3493c893243789b467d6b/
>>  The output url: 
>>
>>  https://testbucket.s3.ap-southeast-1.amazonaws.com/whiteboard/staticConvert/e349fe51afb3493c893243789b467d6b/e349fe51afb3493c893243789b467d6b.qpdf 
>>  ```
>
>2. appOptions is setup only when joining room. So customer needs to make sure the url they generate is valid during the classroom. If it expires, user needs to rejoin so that appOptions which contains urlInterrupter can be triggered again and a new valid pdf file url will be fetched.
>
>3. 'taskid'(customer's own id) needs to be static for the room. When user quit, room state will be stored in our backend. We need this taskid to retrieve the information associated with the pdf file. This id needs to be unique across rooms. The pdfjs information stored in a path created with the scenePath. The scenePath is linked to taskId. Our taskId is unique across the rooms so we recommend using taskId. But if customer can make sure their id is unique, they can also use their own customized id. 
This is important for the case when user wants to review the class after it ends. User needs to input the same 'taskid' to retrieve previous room state.
>
>4. However, the app server still needs to save the taskid and map it to the customized static ID. We will need to check the file conversion task backend log and the whiteboard room log separately based on the taskid and the customized static ID to determine if the issue occurs during the file conversion phase or in the whiteboard room. 

## Troubleshooting

### Failed to fetch pdf.min.mjs

This package downloads the latest PDF.js release from jsDelivr:\
<samp>https://cdn.jsdelivr.net/npm/pdfjs-dist@latest/build/</samp>

To alter this URL or choose a different version, set the app option:

```js
install(register, {
  appOptions: {
    pdfjsLib: 'https://cdn.jsdelivr.net/npm/pdfjs-dist@latest/build/pdf.min.mjs',
    workerSrc: 'https://cdn.jsdelivr.net/npm/pdfjs-dist@latest/build/pdf.worker.min.mjs',
  }
})
```

If the URL is blocked by your [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP),
you have to add them to your `script-src` policy:

```
Content-Security-Policy: script-src https://cdn.jsdelivr.net/
```

Note: If you want to load PDFs on demand (without having to download the entire PDF before rendering),
you need to expose the 'Accept-Ranges' headers on your stored OSS.
If our library cannot obtain this header, it will not make segmented requests.

### TypeError: Promise.withResolvers is not a function

PDF.js uses [`Promise.withResolvers()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/withResolvers), which is a brand new feature in JavaScript.
In case your target environment does not support this, you need to add the following code to the first line of
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

For convenience, you can run `node ./scripts/patch.mjs` in this repo to generate patched `pdf.min.mjs` and `pdf.worker.min.mjs`.
The 2 files will be placed at the `dist` folder.

### Uncaught SyntaxError / Uncaught ReferenceError

This package only targets modern browsers that support [native ES modules](https://caniuse.com/es6-module).

### A certain PDF file opens exceptionally slowly, even several times slower than smaller PDFs.

Most PDF files are designed for easy transmission,
allowing us to render a single page by requesting a portion of the file through Range requests.
However, a few PDF files require a complete download before they can be rendered.
You can check the browser's requests to confirm if the slow rendering of the first page
is due to the complete download of a PDF file.
If so, you need to use [qpdf](https://github.com/qpdf/qpdf) to convert the PDF
into a structure that is easier to transmit.

### What should I do if the OSS I entered in the `Agora Console` is private?

You can add a new property urlInterrupter to the second parameter options when calling register.
This property is a function that returns a publicly accessible address by passing in the URL.

### PDF Rendering Issue

As the package name implies, it uses PDF.js to render the file.
It is possible that PDF.js has a bug when rendering some files.
Please raise an issue [there](https://github.com/mozilla/pdf.js) to ask for help.

## [Changelog](./CHANGELOG.md)

## License

MIT @ [netless](https://github.com/netless-io)
