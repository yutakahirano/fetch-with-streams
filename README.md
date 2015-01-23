Fetch API integrated with Streams
===

This document is about the integration of [the Streams API](https://streams.spec.whatwg.org/) with [the Fetch API](https://fetch.spec.whatwg.org/#fetch-api).
Basically, the integration only adds a way to represent the body data and does not affect the fetch algorithm.

## [Fetch API] (https://fetch.spec.whatwg.org/#fetch-api)

The following should be added in the "Example" section.

```
If you want to receive the body data progressively, use .body attribute.

function consume(stream, total = 0) {
  while (stream.state === "readable") {
    var data = stream.read()
    total += data.byteLength;
    console.log("received " + data.byteLength + " bytes (" + total + " bytes in total).")
  }
  if (stream.state === "waiting") {
    stream.ready.then(() => consume(stream, total))
  }
  return stream.closed
}
fetch("/music/pk/altes-kamuffel.flac")
  .then(res => consume(res.body))
  .then(() => console.log("consumed the entire body without keeping the whole thing in memory!"))
  .catch((e) => console.error("something went wrong", e))
```

## [Bodies] (https://fetch.spec.whatwg.org/#bodies)
A body is a byte stream, which means it is a piped pair of readable and writable byte streams. It has an associated ...(the original description follows)

The __read end__ of a body is the readable stream of the body. It is of type ReadableByteStream.

The __write end__ of a body is the writable stream of the body. It is of type WritableStream.

## [Body mixin](https://fetch.spec.whatwg.org/#body-mixin)

```
callback BodyStreamInit = void (WritableStream);
typedef (Blob or BufferSource or FormData or URLSearchParams or USVString or BodyStreamInit) BodyInit;
```

To extract a byte stream and a 'Content-Type' value from a BodyStreamInit object, run these steps:

1. Let _stream_ be an empty byte stream.
2. Let _Content-Type_ be null.
3. Call _object_ with the _write end_ of the body. Rethrow any exception.
4. Return _stream_ and _Content-Type_.

```
[NoInterfaceObject] interface Body {
    readonly attribute ReadableByteStream body;
    ...
};
```
Objects implementing the Body mixin gain an associated __body__ (a pair of readable and writable byte streams) and a __MIME type__ (initially the empty byte sequence).

The __bodyUsed__ attribute's getter must return if the __read end__ of the body is locked.

The __body__ attribute's getter must return the __read end__ of the body.

Objects implementing the __Body__ mixin also have an associated consume body algorithm, which given a _type_, runs these steps:

1. Let _p_ be a new promise.
2. Let _stream_ be the _read end_ of the body.
3. If _stream_ is locked, reject _p_ with a TypeError.
4. Otherwise, acquire an exclusive lock of _stream_ and run these substeps in parallel: (the original algorithm follows...)

## [Request class] (https://fetch.spec.whatwg.org/#body-mixin)

[Request construction algorithm] (https://fetch.spec.whatwg.org/#dom-request) should be modified as follows.

1. Step 19 is modified as follows.
  - If _r_'s request's mode is _no CORS_, run these substeps:
    1. If _r_'s request's method is not a simple method, throw a TypeError.
    2. If _init_'s body member is present and it is a BodyStreamInit, throw a TypeError.
    3. Set _r_'s Headers object's guard to _request-no-CORS_.
2. A new step is added after Step 19.
  - If _r_'s request's mode is _CORS_, _init_'s body member is present and it is a BodyStreamInit, set _r_'s request's mode to _CORS-with-forced-preflight_.
