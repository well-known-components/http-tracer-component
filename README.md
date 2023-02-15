# HTTP Tracer component

This component uses the [tracer component](https://github.com/well-known-components/tracer-component) over the [HTTP server component](https://github.com/well-known-components/http-server) to build a middleware where each request gets wrapped into a trace span, making it traceable.

The middleware created by the component follows the [Trace Context](https://www.w3.org/TR/trace-context) [W3C Recommendation](https://www.w3.org/standards/types#REC) which defines how by using the `traceparent` and `tracestate` headers, a distributed tracing system can be implemented. It receives the `traceparent` and `tracestate` headers, sets them into the trace span context and then outputs them into the response header, following the specs in the W3C document.

## Usage

The http tracer component is pretty straightforward to use, just initialize the tracer and HTTP server components and then create the HTTP tracer component with them:

```ts
import { createTracerComponent } from '@well-known-components/tracer-component'
import { createHttpTracerComponent } from '@well-known-components/http-tracer-component'

const tracer = createTracerComponent()
const server = await createServerComponent<GlobalContext>({ config, logs }, { cors })
createHttpTracerComponent({ server, tracer })
```

When inside of a handler, the code can access the trace span context at any time by using the [tracer component](https://github.com/well-known-components/tracer-component) API.

To complete the implementation of the [W3C Trace Context](https://www.w3.org/TR/trace-context) tracing mechanism, it is required to propagate the trace headers to any request performed in the service to the other services that are consumed.

There are many ways to do this, but the easiest is to wrap the fetch function to always set the `traceparent` and `tracestate` headers:

```ts
const fetch = (url: string, init?: nodeFetch.RequestInit) => {
  const headers: nodeFetch.HeadersInit = { ...init?.headers }
  const traceParent = tracer.isInsideOfTraceSpan() ? tracer.getTraceChildString() : null
  if (traceParent) {
    ;(headers as { [key: string]: string }).traceparent = traceParent
    const traceState = tracer.getTraceStateString()
    if (traceState) {
      ;(headers as { [key: string]: string }).tracestate = traceState
    }
  }
  return realFetch(url, { ...init, headers })
}
```

As seen in the example above, each time a fetch is performed, the receiving service will receive the `traceparent` and `tracestate` headers. If the receiving services implement the [Trace Context](https://www.w3.org/TR/trace-context) [W3C Recommendation](https://www.w3.org/standards/types#REC), the request will be traceable alongside the services that receive the request.
