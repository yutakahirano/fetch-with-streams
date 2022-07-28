Streaming upload allows web developers to attach a `ReadableStream` to a `Request`.

```js
const rs = new ReadableStream({...});
const resp = await fetch(url, {method: 'POST', body: rs, duplex: 'half'});
```

With this feature, 

 - Web developers can upload unbounded data to the server. This can't be done with
   blobs, because the size (and the contents) of a blob needs to be set before
   starting sending it.
 - Web developers can implement streaming processing easily. Let's assume we have
   a large ReadableStream consisting of Strings (`rs`) and a TransformStream
   (`deflater`) for an awesome compression algorithm - we can make a fetch with the
   compressed stream, like this:
   ```js
   const body = rs.pipeThrough(new TextEncoderStream()).pipeThrough(deflater);
   const resp = await fetch(url, {method: 'POST', body, duplex: 'half'});
   ```
   The encoding and compression is done on-the-fly: we don't need to wait for the
   entire body to be encoded and compressed before starting sending.

Without this feature, web developers typically need to do one (or more) of the
followings to implement the equivalent:

 - Split the stream into multiple chunks and make a fetch for each chunk. This is
   generally bad for performance, and it is difficult to figure out the optimal
   size of each chunk. This also requires additional logic and error handling on
   both of client and server sides.
 - Construct a blob (or ArrayBuffer) to be sent before making a fetch. This will
   impact the latency and peek memory usage heavily.
 - Use protocols such as WebSocket and WebTransport over HTTP/3. The server needs
   to understand these protocols if we choose this option.

These performance shortcomings and cost of additional logic will impact end-users.
In other words, this feature will help end-users enjoy web applications with better
performance, more reliability and less cost.
    
 
There are some caveats:

 - Most of redirects and authentication responses lead to errors. This comes
   from the fact that a streaming body cannot be replayed without storing the
   entire body (even after sending them), and we don't want to store the entire
   body.
 - Only 'cors' and 'same-origin' requests allow streaming upload. You can't use
   streaming upload with 'navigate' and 'no-cors' requests.
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
