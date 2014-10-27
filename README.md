Fetch API integrated with Streams
===

This document is about the integration of [the Streams API](https://streams.spec.whatwg.org/) with [the Fetch API](https://fetch.spec.whatwg.org/#fetch-api).
Basically, the integration only adds a way to represent the body data and does not affect the fetch algorithm.

## [Body mixin](https://fetch.spec.whatwg.org/#body-mixin)

We add WritableStreamRevelation to BodyInit union.

```
callback WritableStreamRevelation = void (WritableStream);
typedef (Blob or BufferSource or FormData or URLSearchParams or USVString or WritableStreamRevelation) BodyInit;
```

## [Request class](https://fetch.spec.whatwg.org/#request-class)




## [Response class](https://fetch.spec.whatwg.org/#response-class)
