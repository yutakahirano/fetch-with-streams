## [Fetch](https://fetch.spec.whatwg.org/) ##

Replace "push" with "enqueue".

Remove __process response body__.

## [Infrastructure](https://fetch.spec.whatwg.org/#infrastructure) ##

To __wait for a response__, if _response_'s body is not null, wait for _response_'s body's stream to be closed or errored.

### [Bodies](https://fetch.spec.whatwg.org/#bodies) ###

A __body__ consists of a ReadableStream object and accompanying data.

A body has an associated stream (a ReadableStream object).

A body has an associated transmitted bytes (an integer). Unless stated it is 0.

A body has an associated total bytes (an integer). Unless stated it is 0.

To clone a body _body_, run these steps:

1. Let «out1, out2» be the result of teeing _body_'s stream. Rethrow any exceptions.
1. Let _newBody_ be a body whose stream is _out2_ and other members are copied from _body_.
1. Set _body_'s stream to _out1_.
1. Return _newBody_.

### [Requests](https://fetch.spec.whatwg.org/#requests) ###

To clone a request _request_, run these steps:

1. Let _newRequest_ be a copy of _request_, except for its body.
1. If _request_'s body is not null, set _newRequest_'s body to the result of cloning _request_'s body. Rethrow any exceptions.
1. Return _newRequest_.

### [Responses](https://fetch.spec.whatwg.org/#responses) ###

To clone a response _response_, run these steps:

1. If _response_ is a filtered response, return a new identical filtered response whose internal response is a clone of _response_'s internal response. Rethrow any exceptions.
1. Let _newResponse_ be a copy of _response_, except for its body.
1. If _response_'s body is not null, set _newResponse_'s body to the result of cloning _response_'s body. Rethrow any exceptions.
1. Return _newResponse_.

### [Body mixin](https://fetch.spec.whatwg.org/#body-mixin) ###

Objects implementing the Body mixin gain an associated body (null or a body, initially null).

To __extract__ a body and a \`Content-Type\` value from _object_, run these steps:

1. Let _stream_ be the result of [constructing a ReadableStream object](https://fetch.spec.whatwg.org/#concept-construct-readablestream). Rethrow any exceptions.
1. Let _Content-Type_ be null.
1. Switch on _object_'s type: (Same as the original, with replacing "push" with "enqueue").
1. Close _stream_.
1. Let _body_ be a body whose stream is _stream_.
1. Return _body_ and _Content-Type_.

...

An Object implementing the Body mixin is said to be __locked__ if the associated body is not null and its stream is locked.

An Object implementing the Body mixin is said to be __disturbed__ if the associated body is not null and its stream is disturbed.

Objects implementing the Body mixin also have an associated __consume data__ algorithm, with given a _type_, runs these steps:

1. If this object is disturbed or locked, return a new promise rejected with a TypeError.
1. Let _stream_ be the associated body's stream, if the associated body is not null, or an empty ReadableStream object otherwise.
1. Let _reader_ be the result of getting a reader from _stream_. If that threw an exception, return a new promise rejected with that exception.
1. Let _promise_ be the result of reading all bytes from stream with _reader_.
1. Return the result of transforming promise by a fulfillment handler that returns the result of the package data algorithm with its first argument, _type_ and __MIME type__.

### [Main fetch](https://fetch.spec.whatwg.org/#main-fetch) ###

(20) Otherwise, if _internalResponse_'s body is non-null, run these substeps:
  1. (Remove)
  1. Once internalResponse's body has been closed or internalResponse has a termination reason, queue a fetch-done task using _request_ and _response_.

### [HTTP-network-or-cache fetch](https://fetch.spec.whatwg.org/#http-network-or-cache-fetch) ###

Replace the first item by the following.

1. Let _httpRequest_ be _request_ if _request_'s window is "no-window" and _request_'s redirect mode is not "follow", and the result of cloning _request_ otherwise. If that threw an exception, abort these steps and return a network error.

### [HTTP-network fetch](https://fetch.spec.whatwg.org/#http-network-fetch) ###

Add the following after the "Read _request_." sentence.

Run the following substeps:

1. Let _strategy_ be an object. The user agent may choose any object.
1. Let _pull_ be an action that resumes the ongoing fetch if it is suspended.
1. Let _cancel_ be an action that terminates the ongoing fetch with reason end-user abort.
1. Let _stream_ be the result of constructing a ReadableStream object with _strategy_, _pull_ and _cancel_.
1. Set _response_'s body to a new body whose stream is _stream_.

The subsubsteps in "Whenever one or more bytes are transmitted, let bytes be the transmitted bytes and run these subsubsteps:" should be the following:

1. (Same)
2. (Same)
3. (Same)
4. Enqueue _bytes_ to _stream_.
5. If _stream_ doesn't need more data and _request_'s synchronous flag is unset, ask the user agent to suspend the ongoing fetch.

### [Request class](https://fetch.spec.whatwg.org/#request-class) ###

In "the Request(input, init) constructor:"

Replace the substeps in the 32th step with the following:

 1. Let _Content-Type_ be null.
 1. Set _temporaryBody_ and _Content-Type_ to the result of extracting a body and \`Content-Type\` value from _init_'s body member. Rethrow any exceptions.
 1. (Same)

Replace the substeps in the 35th step with the following:

 1. Set _input_'s request's body to a new body whose stream is an empty ReadableStream.
 1. Let _reader_ be the result of getting a reader from _stream_.
    Note: This operation will not throw an exception.
 1. Read all bytes from _stream_ with _reader_.
    Note: This operation makes _stream_ disturbed.

The clone() method is defined as follows.

 1. If this Request object is disturbed or locked, throw a TypeError.
 1. Return a new Request object associated with the result of cloning request and a new Headers object whose guard is context object's Headers' guard. Rethrow any exceptions.

### [Response class](https://fetch.spec.whatwg.org/#response-class) ###

Delete the following statements.

- A Response object also has an associated readable stream...
- A Response object is said to be locked...
- A Response object's disturbed predicate...
- A Response object's consume body algorithm,...

The substeps in 7. "If _body_ is given, run these substeps" in Response constructor should be modified as follows.

1. (Same)
1. Let _Content-Type_ be null.
2. Set _r_'s response's body and _Content-Type_ to the result of extracting a body and \`Content-Type\` value from _body_. Rethrow any exceptions.
1. (Same)

The body attribute's getter must return null if the associated body is null and the associated body's stream otherwise.

The clone() method is defined as follows.

1. If this Response object is disturbed or locked, throw a TypeError.
1. Return a new Response object associated with the result of cloning response and a new Headers object whose guard is context object's Headers' guard. Rethrow any exceptions.

### [Fetch method](https://fetch.spec.whatwg.org/#fetch-method) ###

To process response for response, run these substeps:

1. (same)
2. Resolve _p_ with a new Response object associated with _response_ and a new Headers object whose guard is "immutable".

To process response end-of-file for _response_, do nothing.

## [XHR](https://xhr.spec.whatwg.org/) ##

### [The send() method](https://xhr.spec.whatwg.org/#the-send()-method) ###

Add the following subsubsteps at the last of "To process response for _response_, run these subsubsteps:"

- If _response_'s body is null, abort these subsubsteps.
- Let _stream_ be _response_'s body's stream.
- Let _reader_ be the result of getting a reader from _stream_.
 - Note: This operation will not throw an exception.
- Let _read_ be the result of calling ReadFromReadableStreamReader(_reader_).
 - When _read_ is fulfilled with an object whose done property is false and whose value property is a Uint8Array object, run these subsubsubsteps:
  - Append the value property to receivedBytes.
  - If not roughly 50ms have passed since these subsubsteps were last invoked, terminate these subsubsubsteps.
  - Handle errors for _response_.
  - If state is headers received, set state to _loading_ and fire an event named readystatechange.
  - Fire a progress event named progress with _response_'s body's transmitted bytes and _response_'s body's total bytes.
 - When _read_ is fulfilled with an object whose done property is true, run handle response end-of-file for _response_.
 - When _read_ is fulfilled with a value that matches with neither of the above patterns, do nothing.
  - Note: This will not happen.
 - When _read_ is rejected with an error, handle errors for _response_.

Replace the substeps in the 10th step with the following:

1. (Same)
1. (Same)
1. If _response_'s body is null, run handle response end-of-file for _response_ and abort these substeps.
1. Let _reader_ be the result of getting a reader from _stream_.
   Note: This operation will not throw an exception.
1. Let _promise_ be the result of reading all bytes from _response_'s body with _reader_.
1. wait for _promise_ to be resolved or rejected.
 - If _promise_ is resolved with _bytes_, append _bytes_ to receivedBytes and run handle response end-of-file for _response_.
 - If _promise_ is rejected, run handle error for _response_.

To process response end-of-file for _response_, run handle response end-of-file for _response_ if _response_'s body is null.

### [Response](https://xhr.spec.whatwg.org/#xmlhttprequest-response) ###

An XMLHttpRequest object has an associated receivedBytes (a byte sequence). Unless stated otherwise it is an empty byte sequence.

### [Response body](https://xhr.spec.whatwg.org/#response-body) ###

An __arraybuffer response__ is the return value of these steps:

1. (Same)
2. Set response ArrayBuffer object to a new ArrayBuffer object representing receivedBytes and return it.

A __blob response__ is the return value of these steps:

1. (Same)
2. (Same)
3. Set response Blob object to a new Blob object representing receivedBytes and return it.

A __document response__ is the return value of these steps:

1. (Same)
2. If response's body is null, return null.
3. Let _bytes_ be a byte stream containing receivedBytes.
(The original algorithm follows...)

A __JSON response__ is the return value of these steps:
1. (Same)
2. If response's body is null, return null.
3. Let _bytes_ be a byte stream containing receivedBytes.
(The original algorithm follows...)

A __text response__ is the return value of these steps:

1. If response's body is null, return the empty string.
2. Let _bytes_ be a byte stream containing receivedBytes.
(The original algorithm follows...)

