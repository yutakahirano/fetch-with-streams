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

## Somewhere in [Infrastructure] (https://fetch.spec.whatwg.org/#infrastructure) section

To construct a ReadableStream with given _start_, _pull_, _cancel_ and _strategy_ all of which are optional, run these steps.

1. Let _init_ be a new object.
1. Set _init_["start"] to _start_ if _start_ is given.
1. Set _init_["pull"] to _pull_ if _pull_ is given.
1. Set _init_["cancel"] to _cancel_ if _cancel_ is given.
1. Set _init_["strategy"] to _strategy_ if _strategy_ is given.
1. Let _stream_ be the result of calling the initial value of ReadableStream as constructor with _init_ as an argument. Rethrow any exceptions.
1. Return _stream_.

To construct a fixed ReadableStream with given _bytes_, run these steps.

1. Let _enqueue_, _close_ and _error_ be null.
1. Let _start_ be a function that sets _enqueue_, _close_ and _error_ to the first three arguments when called.
1. Let _stream_ be the result of constructing a ReadableStream with _start_. Rethrow any exceptions.
1. Call _enqueue_ with _bytes_. Rethrow any exceptions.
1. Call _close_. Rethrow any exceptions.
1. Return _stream_.

## [Fetching](https://fetch.spec.whatwg.org/#fetching)

Add the following sentences after "To perform a __fetch__ using _request_, ...".

The user agent may be asked to suspend the ongoing fetch. The user agent may either accept or ignore the suspension request.
The suspended fetch can be resumed.

## [Body mixin] (https://fetch.spec.whatwg.org/#body-mixin)

```
[NoInterfaceObject] interface Body {
    readonly attribute ReadableStream body;
    ...
};
```

Objects implementing the Body mixin gain an associated __readable stream__ (initially null), __used flag__ (initially unset), and a __MIME type__ (initially the empty byte sequence).

The body attribute's getter must return the associated readable stream.

Objects implementing the Body mixin also have an associated __consume body__ algorithm, which given a _type_, runs these steps:

1. Let _p_ be a new promise.
2. Let _stream_ be the associated readable stream of the body.
3. If _stream_ is locked, reject _p_ with a TypeError.
4. Otherwise, acquire an exclusive lock of _stream_ and run these substeps in parallel: (the original algorithm follows...)

## [Request class] (https://fetch.spec.whatwg.org/#request-class)

The following item is deleted.

 - A Request object's body is its request body.

[Request construction algorithm] (https://fetch.spec.whatwg.org/#dom-request) should be modified as follows.

1. Step 21 is modified as follows.
  - If _init_'s body member is present, run these substeps:
    1. (same)
    2. Let _bytes_ and _Content-Type_ be the result of extracting _init_'s body member.
    3. Set _r_'s readable stream to the result of constructing a fixed ReadableStream with _bytes_. Rethrow any exceptions.
    4. (same)
1. A new step is added after Step 21.
  - Otherwise, set _r_'s readable stream to the result of constructing a fixed ReadableStream with empty bytes. Rethrow any exceptions.

## [Response class] (https://fetch.spec.whatwg.org/#response-class)

The following item is deleted.

 - A Response object's body is its response body.

[Response construction algorithm] (https://fetch.spec.whatwg.org/#dom-response) should be modified as follows.

1. Step 7 is modified as follow.
  - If _body_ is given, run these substeps:
    1. Let _bytes_ and _Content-Type_ be the result of extracting body.
    2. Set _r_'s readable stream to the result of constructing a fixed ReadableStream with _bytes_. Rethrow any exceptions.
    3. (same)
1. A new step is added after Step 7.
  - Otherwise, set _r_'s readable stream to the result of constructing a fixed ReadableStream with empty bytes. Rethrow any exceptions.

## [Fetch method] (https://fetch.spec.whatwg.org/#fetch-method)

The algorithm is modified as follows.

1. Let _p_ be a new promise.
1. Let _req_ be the result of invoking the initial value of Request as constructor with _input_ and _init_ as arguments. If this throws an exception, reject _p_ with it and return _p_.
1. Let _bytes_ be the result of reading all data stored in _req_'s readable stream.
  - Note: _req_'s readable stream is a ReadableStream created by "constructing a fixed ReadableStream with bytes" operation. Thus it is possible to read all data from _req_'s readable stream synchronously and _req_'s readable stream will automatically become closed once all data is read.
1. Let _request_ be the associated request of _req_.
1. Set _request_'s body to _bytes_.
1. Let _enqueue_, _close_ and _error_ be null.
1. Let _start_ be a function that sets _enqueue_, _close_ and _error_ to the first three arguments when called.
1. Let _strategy_ be a [strategy object] (https://streams.spec.whatwg.org/#queuing-strategy). The user agent may choose any valid strategy object.
1. Let _stream_ be null.
1. Run the following in parallel.
  - Fetch _request_.
  - To process response for _response_, run these substeps:
    1. If _response_'s type is error, reject _p_ with a TypeError and abort these steps.
    1. Let _res_ be a new Response object associated with _response_.
    1. Let _pull_ be a function that resumes the ongoing fetch if it is suspended, when called.
    1. Let _cancel_ be a function that terminates the ongoing fetch algorithm with reason _end-user abort_ when called.
    1. Let _stream_ be the result of constructing a ReadableStream with _start_, _pull_, _cancel_ and _strategy_. If this throws an exception, run the following substeps.
      1. Reject _p_ with that exception.
      1. Terminate the ongoing fetch algorithm with reason _fatal_.
    1. Otherwise, run the following substeps.
      1. Set _res_'s readable stream to _stream_.
      1. Resolve _p_ with _res_.
  - To process response body for _response_, run these substeps:
    1. Let _needsMore_ be the result of calling _enqueue_ with _response_'s body.
       If that throws an exception, terminate the ongoing fetch algorithm with reason _fatal_ and abort these steps.
    1. Clear out _response_'s body.
    1. If _needsMore_ is false, ask the user agent to suspend the ongoing fetch.
  - To process response end-of-file for _response_, call _close_.
1. Return _p_.
