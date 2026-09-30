# Browser-origin checks across HTTP and WebSocket entry points

A local web application exposed state-changing HTTP routes and a streaming WebSocket endpoint. The June review added a browser-origin check at both entry points: an unrelated website should not inherit access merely because its visitor could reach the local application.

The reviewed correction belongs to the historical Python backend; the current application uses Rust.

## Two entry points, two admission decisions

The correction registered middleware for POST, PUT, PATCH and DELETE. Before calling a route, it inspected the supplied Origin and compared its parsed network-location component with the request Host. A mismatch produced HTTP 403. GET, HEAD and OPTIONS passed through that middleware.

WebSocket handling needed an explicit check of its own. The endpoint applied the same comparison before accepting the handshake. A mismatch closed the connection instead of starting a stream. Sharing the comparison function kept HTTP and WebSocket policy aligned without assuming that an HTTP middleware covered the WebSocket upgrade.

The timing matters. Rejecting after accepting a socket would make the streaming connection exist before the admission decision. Placing the check before acceptance separates “may establish this connection” from “what messages will this connection receive.” The historical test demonstrated this distinction by expecting the foreign-origin handshake to fail before a status message arrived.

WebSocket's Origin mechanism exists to let servers reject browser use from unexpected origins. The protocol also explains that a non-browser client can supply a different Origin value; it is not a general client-authentication mechanism. This supports the choice to inspect the handshake while limiting what that inspection proves. [RFC 6455, Origin considerations](https://www.rfc-editor.org/rfc/rfc6455#section-10.2).

## The exact comparison

The helper parsed the Origin, extracted `netloc`, removed trailing dots and lowercased both that value and Host before comparing. Requests with an absent or empty Origin were accepted by this helper. The implementation name suggested same-origin enforcement, but its actual rule was narrower: equality of the compared authority strings.

This illustrative matrix assumes Host is `desk.example.test:8000`. “Passes” means the origin guard permits further processing, not that every route accepts the request.

| Input | Historical guard decision | Reason |
|---|---|---|
| POST with Origin `http://desk.example.test:8000` | Passes | Compared authority matches Host. |
| POST with Origin `http://other.example.test:8000` | HTTP 403 | Different authority. |
| POST with Origin `http://desk.example.test:9000` | HTTP 403 | Different explicit port. |
| POST with Origin `https://desk.example.test:8000` | Passes | The comparison does not include scheme. |
| POST without Origin | Passes | Explicit compatibility path for non-browser callers. |
| GET with an unrelated Origin | Not gated by this middleware | Method is outside the mutation set. |
| WebSocket with an unrelated Origin | Handshake rejected | Endpoint checks before accepting. |
| WebSocket without Origin | Handshake accepted by this guard | Same absent-header policy. |

The standard origin concept includes scheme, host and port. A host:port comparison therefore cannot establish full origin equality, even when it rejects the familiar different-host example. [WHATWG URL origin definition](https://url.spec.whatwg.org/#concept-url-origin).

Python's URL parser provides components; parsing is not itself validation of an application trust policy. The relevant lesson is to name the fields being compared and the missing-header rule explicitly, rather than treating a parsed URL as a security decision. [Python URL parsing security](https://docs.python.org/3/library/urllib.parse.html#url-parsing-security).

## Original patch pattern

The following pseudocode describes the reviewed policy, including its historical limitations. It is not a recommended complete authentication design:

```text
origin_matches_host(origin, host):
    if origin is absent or empty:
        permit
    compare normalized parsed authority with normalized host

HTTP admission:
    for a state-changing method:
        reject if origin_matches_host fails
    continue to route

WebSocket admission:
    reject if origin_matches_host fails
    otherwise accept the handshake and begin streaming
```

Moving the decision into middleware protected new mutation routes that used the same application stack. Keeping a separate WebSocket check addressed the transport boundary rather than relying on a route-specific convention. These were useful coverage improvements even though the underlying policy was deliberately limited.

## What the existing regressions established

The corrective commit added six synthetic cases. HTTP tests used a local test client with a known Host and checked that a foreign Origin returned 403, a matching Origin was not rejected by the middleware, and a request without Origin was likewise not rejected. The positive assertions were “not 403”; they did not claim that the downstream operation completed successfully.

The WebSocket cases checked foreign-origin rejection and successful receipt of an initial status message for matching-Origin and absent-Origin connections. Together, these exercised both entry points and the intentional compatibility path. The test file did not establish scheme-sensitive comparison, a browser exploit reproduction or an authenticated identity.

No-Origin acceptance is a meaningful tradeoff. Scripts and other clients remain compatible, but an omitted header is not evidence that a request came from a trusted script. The rule cannot be used to authenticate a client or authorize a particular data operation. Likewise, Host comparison does not prove which reverse proxy actually transported the request.

The documented June model was local/LAN access without application authentication. Later [authenticated ingress work](authenticated-ingress-and-fixed-control.md) introduced a separate peer, identity and configured-origin boundary. That later design should not be retroactively attributed to this June correction.

## Result and limits

The change rejected supplied foreign-authority origins before mutation routing and before WebSocket acceptance, with existing regressions for the intended admission decisions. Source and those tests were inspected for this retrospective; no browser attack, live service or test suite was exercised. The durable lesson is to assess every transport entry point, then separate browser-origin context, network trust and identity instead of allowing one check to stand in for all three.
