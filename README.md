Fetch API integrated with Streams
===

This document is about the integration of [the Streams API](https://streams.spec.whatwg.org/) with [the Fetch API](https://fetch.spec.whatwg.org/#fetch-api).
Basically, the integration only adds a way to represent the body data and does not affect the fetch algorithm.

## [Bodies] (https://fetch.spec.whatwg.org/#bodies)
A body is a byte stream, which means it is a piped pair of readable and writable byte streams. It has an associated ...(the original description follows)

The __read end__ of a body is the readable stream of the body. It is of type ReadableByteStream.

The __write end__ of a body is the writable stream of the body. It is of type WritableStream.


## [Request] (https://fetch.spec.whatwg.org/#request-class)

```
callback BodyStreamWriter = void (WritableStream);

dictionary RequestInit {
  ByteString method;
  HeadersInit headers;
  BodyInit body;
  BodyStreamWriter bodyWriter;
  RequestMode mode;
  RequestCredentials credentials;
  RequestCache cache;
};
```

The `Request(input, init)` constructor must run these steps:

1. (... existing algorithm steps ...)
2. If _init_'s `bodyWriter` member is present, run these substeps:
   1. Call `bodyWriter` with _r_'s body's __write end__.
3. Otherwise, if _init_'s `body` member is present, run these substeps:
4. (... existing algorithm steps ...)

## [Body mixin](https://fetch.spec.whatwg.org/#body-mixin)

```
[NoInterfaceObject] interface Body {
    readonly attribute ReadableByteStream body;
    ...
};
```
Objects implementing the Body mixin gain an associated __body__ (a pair of readable and writable byte streams) and a __MIME type__ (initially the empty byte sequence). Objects implementing the Body mixin expose methods and attributes of the __read end__ of the body through the __body__ attribute.
The __bodyUsed__ attribute's getter must return if the __read end__ of the body is locked.
The __body__ attribute's getter must return the __read end__ of the body.

Objects implementing the __Body__ mixin also have an associated consume body algorithm, which given a type, runs these steps:

1. Let _p_ be a new promise.
2. Let _stream_ be the _read end_ of the body.
3. If _stream_ is locked, reject _p_ with a TypeError.
4. Otherwise, acquire an exclusive lock of _stream_ and run these substeps in parallel: (the original algorithm follows...)

