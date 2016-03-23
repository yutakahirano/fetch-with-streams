## [Body mixin](https://fetch.spec.whatwg.org/#response-class) ##

To _extract_ a body and a \`Content-Type\` value from _object_, run these steps:

1. (Same)
1. (Same)
1. Switch on _object_'s type:
 - ReadableStream => Set _stream_ to _object_.
 - (The original algorithm follows...)

## [Response class](https://fetch.spec.whatwg.org/#response-class) ##

```
typedef (Blob or BufferSource or FormData or URLSearchParams or USVString or ReadableStream) ResponseBodyInit;

[Construct(ResponseBodyInit, option ResponseIniit init), Exposed=(Window,Worker)]
interface Response {
 ...
}
```
