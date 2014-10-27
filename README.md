Fetch API integrated with Streams
===

This document is about the integration of [the Streams API](https://streams.spec.whatwg.org/) with [the Fetch API](https://fetch.spec.whatwg.org/#fetch-api).
Basically, the integration only adds a way to represent the body data and does not affect the fetch algorithm.

## [Body mixin](https://fetch.spec.whatwg.org/#body-mixin)

We add WritableStreamRevelation to BodyInit union.

```
callback WritableStreamRevelation = void (WritableStream);
dictionary BodyStreamInit {
    DOMString contentType;
    WritableStreamRevelation streamRevelation;
};
typedef (Blob or BufferSource or FormData or URLSearchParams or USVString or BodyStreamInit) BodyInit;
```

To extract a byte stream and a 'Content-Type' value from a BodyStreamInit object, run these steps:
1. Call `object.streamRevelation` with *stream*. Rethrow any exception.
2. Set *Content-Type* to `object.contentType`.

## [Request class](https://fetch.spec.whatwg.org/#request-class)




## [Response class](https://fetch.spec.whatwg.org/#response-class)
