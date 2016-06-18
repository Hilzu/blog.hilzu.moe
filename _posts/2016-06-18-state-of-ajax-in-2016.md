---
layout: post
title: "State of AJAX APIs in 2016"
excerpt: >
  Fetch is the standard solution for doing all sorts of requests from the browser
  using a unified promise based API. Here's how to use it.
date: 2016-06-18 17:37:00 # UTC
---

A bit more than a year ago I was so annoyed with jQuery AJAX API that I created
this blog and [wrote about it][{% post_url 2015-02-10-jquery-ajax-go-away %}].
In the end I mentioned the upcoming [fetch standard][fetch] and thought it's an
interesting option in the future. Well the future is now and it's time to take
another look at the state of AJAX APIs in the year 2016.

*Just use fetch.*

Fetch is the standard solution for doing all sorts of requests from the browser
using a unified promise based API. Fetch already has [good browser support][caniuse]
and the rest of them can be supported with polyfills. The [standard polyfill][fetch-polyfill]
is quite good and works across most modern browsers. If you want to use fetch
in Node you can try [node-fetch][node-fetch]. If you are developing a universal
app take a look at [isomorphic fetch][isomorphic-fetch] or [fetch-ponyfill][fetch-ponyfill]
(ponyfill is a polyfill that doesn't overwrite the native function). Even though
I listed all those options you might just want to use the fetch-ponyfill so
that you can leverage browsers native fetch support.

All said the fetch API still has at least one major drawback that I'm aware of.
Fetch requests [can't be cancelled][fetch-cancel] currently. The feature is going
to be implemented using a standard mechanism for cancelling promises but they
are still just a [proposal][cancelable-promises] and under heavy development. If
you need this feature you're stuck with something else for the time being.

The fetch API is a bit low level for typical web app usage. It doesn't reject
when the response has an 3xx or 4xx error code, only if the request failed on the
network level. I had some wrappers in my previous blog post and now I got some too!

Here's my fetch wrappers from production code with the `checkStatus` function
lifted directly from [GitHub fetch polyfill documentation][fetch-errors].

```javascript
function checkStatus (response) {
  if (response.status >= 200 && response.status < 300) {
    return response
  } else {
    var error = new Error(response.statusText)
    error.response = response
    throw error
  }
}

function fetchJSON (...args) {
  return fetch(...args)
    .then(checkStatus)
    .then((response) => response.json())
}

function postJSON (url, opts = {}) {
  const json = 'application/json'
  opts = Object.assign({}, {
    headers: Object.assign({}, { Accept: json, 'Content-Type': json }, opts.headers),
    method: 'POST'
  }, opts)
  return fetchJSON(url, opts)
}
```

Sometimes you might want to parse the error response body for error messages or
other things. Don't worry, I've got you covered!

```javascript
function checkStatus (response) {
  if (response.status >= 200 && response.status < 300) {
    return response
  } else {
    const error = new Error(response.statusText)
    error.response = response
    return response.json()
      .then((body) => { error.body = body })
      .catch(() => {}) // Catch body parsing errors and continue
      .then(() => { throw error })
  }
}
```

Hope you found this short post interesting and/or helpful. Let me know your
opinion in the comments or in Twitter!

[fetch]: https://fetch.spec.whatwg.org/
[caniuse]: http://caniuse.com/#feat=fetch
[fetch-errors]: https://github.com/github/fetch#handling-http-error-statuses
[fetch-polyfill]: https://github.com/github/fetch
[node-fetch]: https://github.com/bitinn/node-fetch
[fetch-ponyfill]: https://github.com/qubyte/fetch-ponyfill
[isomorphic-fetch]: https://github.com/matthew-andrews/isomorphic-fetch
[fetch-cancel]: https://github.com/whatwg/fetch/issues/27
[cancelable-promises]: https://github.com/domenic/cancelable-promise
