Streaming upload allows web developers to attach a `ReadableStream` to a `Request`.

```js
const rs = ...; // A ReadableStream
const resp = await fetch(url, {method: 'POST', body: rs, duplex: 'half'});
```

With this feature, 

 - You can upload unbound data to the server. This can't be done with blobs,
   because the size (and the contents) of a blob needs to be set before starting
   sending it.
 - You can control the timing of data generation.
 
There are some caveats:

 - Most of redirects and authentication responses lead to errors. This comes
   from the fact that a streaming body cannot be replayed without storing the
   entire body (even after sending them), and we don't want to store the entire
   body.
 - Only 'cors' and 'same-origin' requests allow streaming upload. You can't use
   streaming upload with 'navigation' and 'no-cors' requests.
 - Cross-origin requests with streaming upload always trigger CORS preflight.
 - This feature cannot be used with HTTP/1.x. If the server doesn't support
   HTTP/2 or HTTP/3, the request fails. This is for some compatibility concerns.
   See https://github.com/whatwg/fetch/issues/966 for the past discussions.
 - The browser sends all the request body before starting processing the response
   headers. This, so-called "half-duplex" behavior, is aligned with what browsers
   (Chrome, Firefox, Safari) currently do. There is another, so-called
   "full-duplex" behavior. With the model, user-agents process response
   (including the response body) while sending the request (including the
   request body). This is being discussed at
   https://github.com/whatwg/fetch/issues/1254, but it is not included in the
   first version.
