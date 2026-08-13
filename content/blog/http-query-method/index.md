+++
title = "A New Way to Query HTTP APIs with a Body"
date = "2026-08-13T20:48:39+04:00"
cover = "/blog/http-query-method/images/cover.png"
images = [
    "/blog/http-query-method/images/cover.png"
]
tags = ["HTTP", "API Design", "Golang", "Web Development", "GraphQL"]
keywords = [
    "http query method",
    "rfc 10008",
    "http query method example",
    "post vs query http",
    "http query method golang",
    "http query method vs get",
    "query string too long",
    "graphql http query method",
    "accept-query header",
    "safe idempotent http method",
    "new http method 2026",
    "query method browser support",
    "http query method node.js",
    "http query method nginx"
]
description = "You've probably shipped a POST endpoint that only reads data, because GET can't carry a body and the URL got too long. HTTP finally has a method for that."
showFullContent = false
readingTime = true
+++

Have you done something like this?

```
POST /api/orders/search HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "status": "shipped",
  "createdAfter": "2026-01-01",
  "customer": { "country": "GE" },
  "sortBy": "createdAt",
  "limit": 50
}
```

Most backend developers have. I've seen it in dozens of production APIs. You need to filter, sort, and page through a
resource, the filter shape is too complex for query parameters, so you reach for POST and call the endpoint `/search`.
It works. It also lies about what the request does.

## POST means create, not query

POST is defined to create a subordinate resource or hand data off for processing. It's neither safe nor idempotent by
definition. A client, a proxy, or a cache has no way to know that your `/orders/search` endpoint is actually read-only.
So none of them treat it that way.

That has real consequences, and most of them come down to idempotency. An idempotent request is one you can send twice
and get the same result as sending it once. GET, PUT, and DELETE are defined that way. POST is not, because a POST might
create a resource, and creating the same resource twice is a duplicate, not a no-op.

This is why most API gateways, load balancers, and reverse proxies will not automatically retry a POST request on a
timeout or a dropped connection, even though they will happily retry a GET. The infrastructure has no way to know your
`/orders/search` endpoint only runs a `SELECT`. All it sees is a method that's allowed to mutate state, so the safe
default is to give up and surface the error rather than risk sending the same order twice. You end up writing your own
retry logic in the client, by hand, for a request that never touches a write path in the first place.

Caching has the same problem from a different angle. Browsers and CDNs won't cache a POST response, full stop,
regardless of what headers you attach to it. Every repeated call to your `/search` endpoint hits your backend again,
even if the underlying data hasn't changed since the last request a second ago.

Naming the endpoint `/search` is a workaround. It communicates intent to the next developer reading your code. It
communicates nothing to the HTTP stack, which still sees a method that's allowed to write, and treats it accordingly no
matter what you called the path.

## Why not just use GET?

GET is the semantically correct method for a query. It's safe, it's idempotent, and every cache on the internet already
knows how to handle it. The problem is the body.

The HTTP spec has never technically forbidden a body on a GET request, but it also never defined any semantics for one,
and almost nothing in practice honors it. Browsers strip it, some HTTP libraries drop it silently, some proxies reject
it outright. So in practice, anything you want to send with a GET has to go in the URL.

That's fine for `?status=shipped&limit=50`. It stops being fine once your filter has nested objects, array values, or
free text. You end up either URL-encoding a JSON blob into the query string or flattening it into a pile of
bracket-notation parameters that nobody enjoys writing, reading, or debugging at 2am when a filter silently drops a
field:

```
GET /api/orders?status=shipped&createdAfter=2026-01-01&customer%5Bcountry%5D=GE&sortBy=createdAt&limit=50
```

And URLs have limits. Most servers and proxies cap URL length somewhere around 8,000 characters, some browsers and older
infrastructure cap it much lower. A query with a long list of IDs, a full-text search phrase, or a few nested filters
can hit that ceiling without much effort. When it does, the fix people usually reach for is switching the endpoint to
POST, which brings us right back to the semantics problem.

This isn't a hypothetical. Elasticsearch's `_search` endpoint has accepted `GET` with a request body for years, listed
right alongside `POST /_search` in its own docs as an equivalent way to call it. It's been living with this exact gap
since long before RFC 10008 existed, because a full query DSL document was never going to fit in a query string.

## Enter QUERY

[RFC 10008](https://www.rfc-editor.org/info/rfc10008/) defines a new HTTP method called QUERY, published in June 2026.
It's the first new HTTP method to reach standard status since PATCH in 2010. The RFC's own appendix admits the early
drafts called it SEARCH instead, reusing a method name that WebDAV had already registered back in RFC 5323. They renamed
it to QUERY specifically to avoid dragging in SEARCH's old WebDAV-flavored, XML-only baggage.

QUERY does exactly what the name says, and I mean that as a compliment. Half the fun of naming things in HTTP is that
nobody ever picks the boring, accurate word. It processes a request body and returns a result, the same way POST does,
but it's defined as safe and idempotent, the same way GET is. You get both properties at once, which is the entire
point:

```
QUERY /api/orders HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "status": "shipped",
  "createdAfter": "2026-01-01",
  "customer": { "country": "GE" },
  "sortBy": "createdAt",
  "limit": 50
}
```

Same payload as the POST example at the top of this post. The difference is what the method declares about it.

### What safe and idempotent actually buy you

Because QUERY is defined as idempotent, the same infrastructure that refuses to auto-retry a POST is free to retry a
QUERY on a timeout, since sending it twice is guaranteed to produce the same result as sending it once. Because it's
defined as safe, a cache is allowed to store the response and serve it back on an identical request without hitting your
backend again. None of that requires the gateway or the CDN to inspect your body or know anything about orders. The
method itself carries the guarantee.

### Accept-Query and the body format

The RFC also defines `Accept-Query`, a response header a server can use to advertise which media types it accepts in a
QUERY body, similar to how `Accept` works in reverse. A server can return `Accept-Query: application/json,
application/x-www-form-urlencoded` on an `OPTIONS` response so a client knows what formats are supported before sending
the actual query.

The body format itself is entirely up to you. JSON is the obvious default for most web APIs, but the RFC's own examples
advertise `application/sql` and `application/xslt+xml` right alongside `application/jsonpath` in `Accept-Query`, just to
make the point that QUERY doesn't care what's inside the body. If your query language of choice already has a media
type, QUERY carries it fine.

### Turning a query into a bookmarkable resource

A server can answer a QUERY with a `Location` header pointing at a resource that represents that exact query, something
like `Location: /stored-queries/4815162342`. From then on, a client can just `GET` that URL directly to rerun the same
query, no need to resend the body at all. It's an opt-in the server has to build, not something QUERY gives you for
free, but it's a clean way to turn a one-off query into something bookmarkable and cacheable by URL.

## A natural fit for GraphQL

GraphQL has been living with this exact mismatch since it launched. A GraphQL query is read-only by design, but the spec
sends it over POST, because a query document doesn't reliably fit in a URL. Every GraphQL query today technically looks
like a write to anything inspecting it at the HTTP layer, even though the server never touches persistent state for it.

QUERY maps onto this cleanly. A GraphQL `query` operation is safe and idempotent, exactly what QUERY is defined for. A
`mutation` isn't, and should keep using POST. Splitting the two across methods that actually match their semantics means
caches and gateways can finally tell the difference between a GraphQL read and a GraphQL write without parsing the
request body to find out. Nothing in the GraphQL spec requires POST specifically, so this is a transport-layer change a
server could adopt without touching the query language itself.

## Who supports it today

Server-side adoption is ahead of where you'd expect for a method that's a couple months old.

- **Node.js** has recognized QUERY as a valid HTTP method since 22.2.0, released in June 2024, two years before the RFC
  was finalized.
- **OpenAPI 3.2**, released in September 2025, documents QUERY as a first-class operation type alongside GET, POST, and
  the rest.
- **nginx** doesn't have it yet. There's an open pull request adding QUERY support to core, but as of this writing it's
  still waiting on review, which for nginx isn't unusual and isn't a signal that anything's wrong with it.
- Framework support is catching up unevenly. Some have open pull requests, others are still discussing it. Expect gaps
  for a while if you're on a framework that hasn't shipped explicit support yet.

## Browser support is the rough edge

This is where you should slow down before shipping QUERY on a public-facing API.

The Fetch API doesn't forbid QUERY as a method token, so `fetch("/api/orders", { method: "QUERY", body: ... })` works
today in current browsers for same-origin requests. The catch is CORS. QUERY isn't one of the safelisted methods (GET,
POST, HEAD), so any cross-origin request using it triggers a preflight `OPTIONS` request, and your server has to respond
with `Access-Control-Allow-Methods` including `QUERY` for the real request to go through.

That's not a blocker, preflight is a solved problem for plenty of methods already. But it does mean QUERY isn't a
drop-in replacement you can flip on without touching your CORS configuration, and older browsers or strict corporate
proxies may not have caught up yet. For server-to-server calls, none of this matters, and I wouldn't lose sleep over it
there. For a public API called directly from browser JavaScript, test it before you rely on it, and keep the POST
fallback around a little longer than you think you need to.

## A Go server example

Go's standard library doesn't have an `http.MethodQuery` constant yet, but `net/http` has supported registering handlers
for arbitrary method verbs since the routing improvements in Go 1.22. You don't need a third-party router to handle
QUERY:

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
)

type orderQuery struct {
	Status       string `json:"status"`
	CreatedAfter string `json:"createdAfter"`
	Limit        int    `json:"limit"`
}

func searchOrders(w http.ResponseWriter, r *http.Request) {
	var q orderQuery
	if err := json.NewDecoder(r.Body).Decode(&q); err != nil {
		http.Error(w, "invalid query body", http.StatusBadRequest)
		return
	}
	defer r.Body.Close()

	results := findOrders(q)

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(results)
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("QUERY /api/orders", searchOrders)

	log.Println("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

The `"QUERY /api/orders"` pattern is the same method-plus-path syntax `ServeMux` already supports for GET, POST, and
anything else. Nothing special about QUERY here, it's just a method string the router happens to match.

Run that and you can hit it straight from curl, no client code needed:

```bash
curl -X QUERY http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"status": "shipped", "limit": 50}'
```

curl has never restricted which method you pass to `-X`, so this works today regardless of what your language of choice
has or hasn't caught up on.

Calling it from a Go client works the same way any other method does, `http.NewRequest` doesn't restrict which verbs you
can use:

```go
req, err := http.NewRequest("QUERY", "http://localhost:8080/api/orders", body)
req.Header.Set("Content-Type", "application/json")

resp, err := http.DefaultClient.Do(req)
```

There's an [open proposal to add a `http.MethodQuery` constant](https://github.com/golang/go/issues/80058) to the
standard library, following the same pattern as `http.MethodPatch`. Until it lands, the literal string works exactly the
same way.

## Where this leaves you

If you're building an internal API and your current `/search` endpoints are all POST, QUERY is a clean semantic upgrade
with no browser CORS concerns to worry about, since server-to-server calls don't hit preflight. If you're exposing a
public API that browsers call directly, QUERY is usable today but budget time for CORS configuration and don't assume
every client library out there has caught up.

I'd start with the internal case. Ship it there, watch how your gateway and cache actually behave with it, then decide
whether the CORS work makes sense for the public side. Either way, keep this method in mind the next time you catch
yourself naming an endpoint `/search` and reaching for POST out of habit.
