Fetch API integrated with Streams
===

This document is about the integration of [the Streams API](https://streams.spec.whatwg.org/) with [the Fetch API](https://fetch.spec.whatwg.org/#fetch-api).
Basically, the integration only adds a way to represent the body data and does not affect the fetch algorithm.

## Global

Replace every "push" word on request / response body with "enqueue".

## [Fetch API] (https://fetch.spec.whatwg.org/#fetch-api)

The following should be added in the "Example" section.

```
If you want to receive the body data progressively, use .body attribute.

function consume(reader, total = 0) {
  return reader.read().then(({done, value}) => {
    if (done) {
      return
    }
    total += value.byteLength
    console.log("received " + value.byteLength + " bytes (" + total + " bytes in total).")
    return consume(reader, total)
  })
}

fetch("/music/pk/altes-kamuffel.flac")
  .then(res => consume(res.body.getReader()))
  .then(() => console.log("consumed the entire body without keeping the whole thing in memory!"))
  .catch((e) => console.error("something went wrong", e))
```

## Somewhere in [Infrastructure] (https://fetch.spec.whatwg.org/#infrastructure) section

### ReadableByteStream class
```
[NoInterfaceObject]
interface ReadableByteStreamReader {
  readonly attribute Promise<void> closed;
  readonly Promise<any> read();
  void cancel();
};

interface ReadableByteStream {
  ReadableByteStreamReader getReader();
  void cancel();
};
```

ReadableByteStream and ReadableByteStreamReader are just aliases of ReadableStream and ReadableStreamReader, respectively.

__NOTE:__ the Streams Standard will define these classes in the future, but it does not yet define them; see [whatwg/streams#300](https://github.com/whatwg/streams/issues/300). However, `ReadableByteStream` will be a superset (in both behavior and API) of the already-defined [`ReadableStream` class](https://streams.spec.whatwg.org/#rs-class). In what follows, we treat `ReadableByteStream` as a `ReadableStream` for now.

To __construct a ReadableByteStream__ with given _start_, _pull_, _cancel_ and _strategy_ all of which are optional, run these steps.

1. Let _init_ be a new object.
1. Set _init_["start"] to _start_ if _start_ is given.
1. Set _init_["pull"] to _pull_ if _pull_ is given.
1. Set _init_["cancel"] to _cancel_ if _cancel_ is given.
1. Set _init_["strategy"] to _strategy_ if _strategy_ is given.
1. Let _stream_ be the result of calling the initial value of ReadableByteStream as constructor with _init_ as an argument. Rethrow any exceptions.
1. Return _stream_.

To __construct a fixed ReadableByteStream__ with given _chunks_, run these steps.

1. Let _stream_ be the result of constructing a ReadableByteStream. Rethrow any exceptions.
1. For each _chunk_ in _chunks_, run these substeps:
  1. Call [EnqueueInReadableStream](https://streams.spec.whatwg.org/#enqueue-in-readable-stream)(_stream_, _chunk_). Rethrow any exceptions.
1. Call [CloseReadableStream](https://streams.spec.whatwg.org/#close-readable-stream)(_stream_). Rethrow any exceptions.
1. Return _stream_.

An __empty ReadableByteStream__ is the result of constructing a fixed ReadableByteStream with an empty array.

Note: constructing an empty ReadableByteStream must not throw an exception.

A ReadableByteStream _stream_ is said to be __readable__ if _stream_@[[state]] is "readable".

A ReadableByteStream _stream_ is said to be __closed__ if _stream_@[[state]] is "closed".

A ReadableByteStream _stream_ is said to be __errored__ if _stream_@[[state]] is "errored".

## [Fetching](https://fetch.spec.whatwg.org/#fetching)

Add _garbage collection_ as one of fetch termination reasons.

Add the following sentences after "To perform a __fetch__ using _request_, ...".

The user agent may be asked to suspend the ongoing fetch. The user agent may either accept or ignore the suspension request.
The suspended fetch can be resumed.

The user agent should ignore the suspension request if the ongoing fetch is updating the response in the HTTP cache for the request.

Note: The user agent must not update the entry in the HTTP cache for a request if request's cache mode is "no-store".

Note: The user agent must not update the entry in the HTTP cache for a request if "Cache-Control: no-cache" HTTP header appears in the response. [RFC7234] (http://tools.ietf.org/html/rfc7234)

## [Body mixin] (https://fetch.spec.whatwg.org/#body-mixin)

Objects implementing the Body mixin gain an associated __passed flag__ (initially unset) and a __MIME type__ (initially the empty byte sequence).

Objects implementing the Body mixin must define an associated __consume body__ operation.

Objects implementing the Body mixin must define an associated __used__ predicate.

The __bodyUsed__ attribute's getter must return true if the body is used and false otherwise.

Objects implementing the Body mixin also have an associated __package body data__ algorithm, which given _bytes_, a _type_ and a _MIME type_, runs these steps:

1. Switch on _type_:
  - ArrayBuffer
    1. Return an ArrayBuffer whose contents are _bytes_.
  - Blob
    1. Return a Blob whose contents are _bytes_ and type is _MIME type_.
  - FormData
    1. If _MIME type_ (ignoring parameters) is \`multipart/form-data\`, run these substeps:
      1. Parse _bytes_, using the value of the \`boundary\` parameter from _MIME type_ and [utf-8] (https://encoding.spec.whatwg.org/#utf-8) as encoding, per the rules set forth in _Returning Values from Forms: multipart/form-data_. [\[RFC2388\]] (https://fetch.spec.whatwg.org/#refsRFC2388)
      1. If that fails for some reason, throw a TypeError.
      1. Return a new FormData object, running [create an entry] (https://xhr.spec.whatwg.org/#create-an-entry) for each entry as the result of parsing operation and appending it to [entries] (https://xhr.spec.whatwg.org/#concept-formdata-entry).
      1. (same Note)
    1. Otherwise, if _MIME type_ (ignoring parameters) is \`application/x-www-form-urlencoded\`, run these substeps:
      1. Let _entries_ be the result of [parsing] (https://url.spec.whatwg.org/#concept-urlencoded-parser) _bytes_.
      1. If _entries_ is failure, throw a TypeError.
      1. Return a new FormData object whose [entries] (https://xhr.spec.whatwg.org/#concept-formdata-entry) are _entries_.
    1. Otherwise, throw a TypeError.
  - JSON
    1. Return the result of invoking the initial value of the parse property of the JSON object with the result of running [utf-8 decode] (https://encoding.spec.whatwg.org/#utf-8-decode) on _bytes_ as argument. Rethrow any exceptions.
  - text
    1. Return the result of running [utf-8 decode] (https://encoding.spec.whatwg.org/#utf-8-decode) on _bytes_.

## [Request class] (https://fetch.spec.whatwg.org/#request-class)

The following item is deleted.

 - A Request object's body is its request body.

Add the following:

A Request has an associated __locked flag__ (initially unset).

A Request has an associated __used__ predicate that returns true if one of the following holds and false otherwise:

- passed flag is set.
- locked flag is set.

Request's associated __consume body__ algorithm, which given a _type_, runs these steps:

1. If passed flag is set or locked flag is set, return a new promise rejected with a TypeError.
1. Set locked flag.
1. Let _p_ be a new promise.
1. Run these substeps in parallel.
  1. Resolve _p_ with the result of running package body data with request body, _type_ and the associated MIME type. If that threw an exception, reject _p_ with that exception.
  1. Clear out request body.
  1. Unset locked flag.
1. Return _p_.

[Request construction algorithm] (https://fetch.spec.whatwg.org/#dom-request) should be modified as follows.

1. Step 2 is modified as follows.
  - If _input_ is a Request object and _input_'s body is non-null, run these substeps:
    1. If _input_ is used, throw a TypeError.
    1. [Same]
1. Step 25 is modified as follows.
  - If _input_ is a Request object and _input_'s body is non-null, run these substeps:
    1. [Same]
    1. Set _input_'s passed flag.

[clone() method on Request] (https://fetch.spec.whatwg.org/#dom-request-clone) should be modified as follows.

1. Step1 is modified as follows.
  - If passed flag is set or locked flag is set, throw a TypeError.

## [Response class] (https://fetch.spec.whatwg.org/#response-class)

```
interface Response {
  ...
  ReadableByteStream body;
};
```

A Response object has an associated __readable stream__ (initially an empty ReadableByteStream). Each chunk in this stream must be a Uint8Array.

A Response object has an associated __used__ predicate that returns true if one of the following holds:

- passed flag is set.
- The associated readable stream is [locked to a reader](https://streams.spec.whatwg.org/#is-readable-stream-locked).

The __body__ attribute's getter must return the associated readable stream.

Response's associated __consume body__ algorithm, which given a _type_, runs these steps:

1. If passed flag is set, return a new promise rejected with a TypeError.
1. Let _stream_ be the associated readable stream.
1. Let _reader_ be the result of running [acquiring an exclusive stream reader](https://streams.spec.whatwg.org/#acquire-exclusive-stream-reader) for _stream_. If that threw an exception, return a promise rejected with that exception.
1. Let _p_ be a new promise.
1. Run these substeps in parallel.
  1. Let _chunks_ be the result of using implementation-specific mechanisms to repeatedly read from _stream_, using the exclusive access granted by _reader_, until _stream_ becomes closed or errored.
  1. Let _bytes_ be the concatenation of each chunk's contents.
    - Note: Each chunk must be a Uint8Array.
  1. If _stream_ is errored, reject _p_ with _stream_@[[storedError]].
  1. Otherwise, resolve _p_ with the result of running package body data with _bytes_, _type_ and the associated MIME type. If that threw an exception, reject _p_ with that exception.
1. Return _p_.

The following item is deleted.

 - A Response object's body is its response body.

[Response construction algorithm] (https://fetch.spec.whatwg.org/#dom-response) should be modified as follows.

- Step 7 is modified as follow.
  - If _body_ is given, run these substeps:
    1. Let _bytes_ and _Content-Type_ be the result of extracting body.
    1. Set _r_'s readable stream to the result of constructing a fixed ReadableByteStream with an array consisting of a Uint8Array whose contents are _bytes_. Rethrow any exceptions.
    1. (same)

[clone() method on Response] (https://fetch.spec.whatwg.org/#dom-response-clone) should be modified as follows.

1. If passed flag is set, throw a TypeError.
1. Let _«out1, out2»_ be the result of invoking [TeeReadableStream] (https://streams.spec.whatwg.org/#tee-readable-stream) with the associated readable stream and __true__. Rethrow any exceptions.
1. Let _newResponse_ be a copy of response, except that _newResponse_'s body is an empty bytes.
1. Let _r_ be a new Response object associated with _newResponse_ and a new Headers object whose guard is context object's Headers' guard.
1. Set the associated readable stream to _out1_.
1. Set _r_'s associated readable stream to _out2_.
1. Return _r_.

## [Fetch method] (https://fetch.spec.whatwg.org/#fetch-method)

The algorithm is modified as follows.

1. Let _p_ be a new promise.
1. Let _req_ be the result of invoking the initial value of Request as constructor with _input_ and _init_ as arguments. If that threw an exception, reject _p_ with it and return _p_.
1. Let _request_ be the associated request of _req_.
1. Let _strategy_ be a [strategy object] (https://streams.spec.whatwg.org/#queuing-strategy). The user agent may choose any valid strategy object.
1. Let _stream_ be null.
1. Run the following in parallel.
  - Fetch _request_.
  - To process response for _response_, run these substeps:
    1. If _response_'s type is error, reject _p_ with a TypeError and abort these steps.
    1. Let _res_ be a new Response object associated with _response_.
    1. Let _pull_ be a function that resumes the ongoing fetch if it is suspended, when called.
    1. Let _cancel_ be a function that terminates the ongoing fetch algorithm with reason _end-user abort_ when called.
    1. Let _stream_ be the result of constructing a ReadableByteStream with _pull_, _cancel_ and _strategy_. If that threw an exception, run the following substeps.
      1. Reject _p_ with that exception.
      1. Terminate the ongoing fetch algorithm with reason _fatal_.
    1. Otherwise, run the following substeps.
      1. Set _res_'s readable stream to _stream_.
      1. Resolve _p_ with _res_.
  - To process response body for _response_, run these substeps:
    1. Let _needsMore_ be the result of [EnqueueInReadableStream](https://streams.spec.whatwg.org/#enqueue-in-readable-stream)(_response_'s readable stream, a Uint8Array whose contents are _response_'s body).
    1. Clear out _response_'s body.
    1. If _needsMore_ is false, ask the user agent to suspend the ongoing fetch.
  - To process response end-of-file for _response_, call [CloseReadableStream](https://streams.spec.whatwg.org/#close-readable-stream)(_response_'s readable stream).
1. Return _p_.

### Fetch termination initiated by the user agent

The user agent may terminate an ongoing fetch with reason _garbage collection_ if that termination is not observable through script.

Note: The server observing garbage collection has precedent, e.g. with WebSocket and XMLHttpRequest.

(Example)

```
// The user agent can terminate the fetch because the termination cannot
// be observed.
fetch("http://www.example.com/")

// The user agent cannot terminate the fetch because the termination can be
// observed through the promise.
window.promise = fetch("http://www.example.com/")

// The user agent can terminate the fetch because the associated body
// is not observable.
window.promise = fetch("http://www.example.com/").then(res => res.headers)

// The user agent can terminate the fetch because the termination cannot
// be observed.
fetch("http://www.example.com/").then(res => res.body.getReader().closed)

// The user agent cannot terminate the fetch because one can observe the
// termination by registering a handler for the promise object.
window.promise = fetch("http://www.example.com/").then(res =>
  res.body.getReader().closed)

// The user agent cannot terminate the fetch as termination would be observable
// via the registered handler.
fetch("http://www.example.com/").then(res => {
  res.body.getReader().closed.then(() => console.log("stream closed!"))
})
```
